# IPFS(四) 源码解读之-p2p

```go
package p2p

import (
   "context"
   "errors"
   "time"

   net "gx/ipfs/QmPjvxTpVH8qJyQDnxnsxF9kv9jezKD1kozz1hs3fCGsNh/go-libp2p-net"
   manet "gx/ipfs/QmV6FjemM1K8oXjrvuq3wuVWWoU2TLDPmNnKrxHzY3v6Ai/go-multiaddr-net"
   ma "gx/ipfs/QmYmsdtJ3HsodkePE3eU3TsCaP2YvPZJ4LoXnNkDE5Tpt7/go-multiaddr"
   pro "gx/ipfs/QmZNkThpqfVXs9GNbexPrfBbXSLNYeKrE7jwFM2oqHbyqN/go-libp2p-protocol"
   pstore "gx/ipfs/QmZR2XWVVBCtbgBWnQhWk2xcQfaR3W8faQPriAiaaj7rsr/go-libp2p-peerstore"
   p2phost "gx/ipfs/Qmb8T6YBBsjYsVGfrihQLfCJveczZnneSBqBKkYEBWDjge/go-libp2p-host"
   peer "gx/ipfs/QmdVrMn1LhB4ybb8hMVaMLXnA8XRSewMnK6YqXKXoTcRvN/go-libp2p-peer"
)
//P2P结构保存当前正在运行的流/监听器的信息
// P2P structure holds information on currently running streams/listeners
type P2P struct {
    //监听器
   Listeners ListenerRegistry
    //数据流
   Streams   StreamRegistry
	//节点ID
   identity  peer.ID
    //节点地址
   peerHost  p2phost.Host
    //一个线程安全的对等节点存储
   peerstore pstore.Peerstore
}
//创建一个新的p2p结构
// NewP2P creates new P2P struct
//这个新的p2p结构不包含p2p结构中的监听器和数据流
func NewP2P(identity peer.ID, peerHost p2phost.Host, peerstore pstore.Peerstore) *P2P {
   return &P2P{
      identity:  identity,
      peerHost:  peerHost,
      peerstore: peerstore,
   }
}
//新建一个数据流  工具方法 构建一个有节点id，内容和协议的流
func (p2p *P2P) newStreamTo(ctx2 context.Context, p peer.ID, protocol string) (net.Stream, error) {
    //30s 后会自动timeout
   ctx, cancel := context.WithTimeout(ctx2, time.Second*30) //TODO: configurable?
   defer cancel()
   err := p2p.peerHost.Connect(ctx, pstore.PeerInfo{ID: p})
   if err != nil {
      return nil, err
   }

    //NewStream为给定的对等点p打开一个新的流，并编写一个p2p/协议 带有给定协议. id的头文件。
   return p2p.peerHost.NewStream(ctx2, p, pro.ID(protocol))
}
//对话为远程监听器创建新的P2P流
//创建一个新的p2p流实现对对话的监听
// Dial creates new P2P stream to a remote listener
//Multiaddr是一种跨协议、跨平台的表示格式的互联网地址。它强调明确性和自我描述。
//对内接收
func (p2p *P2P) Dial(ctx context.Context, addr ma.Multiaddr, peer peer.ID, proto string, bindAddr ma.Multiaddr) (*ListenerInfo, error) {
    //获取一些节点信息   network, host, nil
   lnet, _, err := manet.DialArgs(bindAddr)
   if err != nil {
      return nil, err
   }
	//监听信息
   listenerInfo := ListenerInfo{
      	//节点身份
      Identity: p2p.identity,
        ////应用程序协议标识符。
      Protocol: proto,
   }
	//调用newStreamTo 通过ctx（内容） peer（节点id） proto（协议标识符） 参数获取一个新的数据流 
   remote, err := p2p.newStreamTo(ctx, peer, proto)
   if err != nil {
      return nil, err
   }
	//network协议标识
   switch lnet {
       //network为"tcp", "tcp4", "tcp6"
   case "tcp", "tcp4", "tcp6":
       //从监听器获取新的信息  nla.Listener, nil
      listener, err := manet.Listen(bindAddr)
      if err != nil {
         if err2 := remote.Reset(); err2 != nil {
            return nil, err2
         }
         return nil, err
      }
		//将获取的新信息保存到listenerInfo
      listenerInfo.Address = listener.Multiaddr()
      listenerInfo.Closer = listener
      listenerInfo.Running = true
		//开启接受
      go p2p.doAccept(&listenerInfo, remote, listener)

   default:
      return nil, errors.New("unsupported protocol: " + lnet)
   }

   return &listenerInfo, nil
}
//
func (p2p *P2P) doAccept(listenerInfo *ListenerInfo, remote net.Stream, listener manet.Listener) {
    //关闭侦听器并删除流处理程序
   defer listener.Close()
	//Returns a Multiaddr friendly Conn
    //一个有好的 Multiaddr 连接
   local, err := listener.Accept()
   if err != nil {
      return
   }

   stream := StreamInfo{
       //连接协议
      Protocol: listenerInfo.Protocol,
		//定位节点
      LocalPeer: listenerInfo.Identity,
       //定位节点地址
      LocalAddr: listenerInfo.Address,
		//远程节点
      RemotePeer: remote.Conn().RemotePeer(),
       //远程节点地址
      RemoteAddr: remote.Conn().RemoteMultiaddr(),
		//定位
      Local:  local,
       //远程
      Remote: remote,
		//注册码
      Registry: &p2p.Streams,
   }
	//注册连接信息
   p2p.Streams.Register(&stream)
    //开启节点广播
   stream.startStreaming()
}
//侦听器将流处理程序包装到侦听器中
// Listener wraps stream handler into a listener
type Listener interface {
   Accept() (net.Stream, error)
   Close() error
}
//P2PListener保存关于侦听器的信息
// P2PListener holds information on a listener
type P2PListener struct {
   peerHost p2phost.Host
   conCh    chan net.Stream
   proto    pro.ID
   ctx      context.Context
   cancel   func()
}
//等待侦听器的连接
// Accept waits for a connection from the listener
func (il *P2PListener) Accept() (net.Stream, error) {
   select {
   case c := <-il.conCh:
      return c, nil
   case <-il.ctx.Done():
      return nil, il.ctx.Err()
   }
}
 //关闭侦听器并删除流处理程序
// Close closes the listener and removes stream handler
func (il *P2PListener) Close() error {
   il.cancel()
   il.peerHost.RemoveStreamHandler(il.proto)
   return nil
}
// Listen创建新的P2PListener
// Listen creates new P2PListener
func (p2p *P2P) registerStreamHandler(ctx2 context.Context, protocol string) (*P2PListener, error) {
   ctx, cancel := context.WithCancel(ctx2)

   list := &P2PListener{
      peerHost: p2p.peerHost,
      proto:    pro.ID(protocol),
      conCh:    make(chan net.Stream),
      ctx:      ctx,
      cancel:   cancel,
   }

   p2p.peerHost.SetStreamHandler(list.proto, func(s net.Stream) {
      select {
      case list.conCh <- s:
      case <-ctx.Done():
         s.Reset()
      }
   })

   return list, nil
}
// NewListener创建新的p2p侦听器
// NewListener creates new p2p listener
//对外广播
func (p2p *P2P) NewListener(ctx context.Context, proto string, addr ma.Multiaddr) (*ListenerInfo, error) {
    //调用registerStreamHandler 构造一个新的listener
   listener, err := p2p.registerStreamHandler(ctx, proto)
   if err != nil {
      return nil, err
   }
	//构造新的listenerInfo
   listenerInfo := ListenerInfo{
      Identity: p2p.identity,
      Protocol: proto,
      Address:  addr,
      Closer:   listener,
      Running:  true,
      Registry: &p2p.Listeners,
   }

   go p2p.acceptStreams(&listenerInfo, listener)
	//注册连接信息
   p2p.Listeners.Register(&listenerInfo)

   return &listenerInfo, nil
}
//接受流
func (p2p *P2P) acceptStreams(listenerInfo *ListenerInfo, listener Listener) {
   for listenerInfo.Running {
          //一个有好的 远程 连接
      remote, err := listener.Accept()
      if err != nil {
         listener.Close()
         break
      }

      local, err := manet.Dial(listenerInfo.Address)
      if err != nil {
         remote.Reset()
         continue
      }

      stream := StreamInfo{
         Protocol: listenerInfo.Protocol,

         LocalPeer: listenerInfo.Identity,
         LocalAddr: listenerInfo.Address,

         RemotePeer: remote.Conn().RemotePeer(),
         RemoteAddr: remote.Conn().RemoteMultiaddr(),

         Local:  local,
         Remote: remote,

         Registry: &p2p.Streams,
      }
		//注册数据流
      p2p.Streams.Register(&stream)
       //数据流广播
      stream.startStreaming()
   }
    //取消注册表中的p2p侦听器
   p2p.Listeners.Deregister(listenerInfo.Protocol)
}
// CheckProtoExists检查是否注册了协议处理程序
// mux处理程序
// CheckProtoExists checks whether a protocol handler is registered to
// mux handler
func (p2p *P2P) CheckProtoExists(proto string) bool {
   protos := p2p.peerHost.Mux().Protocols()

   for _, p := range protos {
      if p != proto {
         continue
      }
      return true
   }
   return false
}
```
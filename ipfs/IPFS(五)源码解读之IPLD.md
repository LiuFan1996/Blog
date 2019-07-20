```
package format

import (
   "context"
   "fmt"

   blocks "gx/ipfs/QmVzK524a2VWLqyvtBeiHKsUAWYgeAk4DBeZoY7vpNPNRx/go-block-format"

   cid "gx/ipfs/QmYVNvtQkeZ6AKSwDrjQTs432QtL6umrrK41EBq3cu7iSP/go-cid"
)
//解析器接口
type Resolver interface {
	//解析通过该节点的路径，在任何链接边界处停止
	//返回找到的对象以及要遍历的剩余路径
   // Resolve resolves a path through this node, stopping at any link boundary
   // and returning the object found as well as the remaining path to traverse
   Resolve(path []string) (interface{}, []string, error)
	
	//树列出了“路径”下对象内的所有路径，以及给定深度。
	//要列出整个对象(类似于“find .”)，传递“”和-1
   // Tree lists all paths within the object under 'path', and up to the given depth.
   // To list the entire object (similar to `find .`) pass "" and -1
   Tree(path string, depth int) []string
}

//Node是所有IPLD节点必须实现的基本接口。
//节点是不可变的，接口上定义的所有方法都是**不可变的
/ / 线程安全的。
// Node is the base interface all IPLD nodes must implement.
//
// Nodes are **Immutable** and all methods defined on the interface are
// **Thread Safe**.
type Node interface {
   blocks.Block
   Resolver
	
	// ResolveLink是一个助手函数，它调用resolve并断言
	//输出是一个链接
   // ResolveLink is a helper function that calls resolve and asserts the
   // output is a link
   ResolveLink(path []string) (*Link, []string, error)
	//返回副本的深度
   // Copy returns a deep copy of this node
   Copy() Node
	
	//返回对象中所有的链接
   // Links is a helper function that returns all links within this object
   Links() []*Link

   // TODO: not sure if stat deserves to stay
   Stat() (*NodeStat, error)
   
	//Size返回序列化对象的字节大小
   // Size returns the size in bytes of the serialized object
   Size() (uint64, error)
}

//链路表示节点之间的IPFS Merkle DAG链路。
// Link represents an IPFS Merkle DAG Link between Nodes.
type Link struct {
	//utf字符串的名字。每个对象应该是唯一的
   // utf string name. should be unique per object
   Name string // utf8
	//目标对象的累积大小
   // cumulative size of target object
   Size uint64
	//目标对象的多哈希
   // multihash of the target object
   Cid *cid.Cid
}

//NodeStat是节点的统计对象。主要尺寸。
// NodeStat is a statistics object for a Node. Mostly sizes.
type NodeStat struct {
   Hash           string
   NumLinks       int // number of links in link table 链接表中的链接数
   BlockSize      int // size of the raw, encoded data 原始编码数据的大小
   LinksSize      int // size of the links segment 链接段的大小
   DataSize       int // size of the data segment 数据段的大小
   CumulativeSize int // cumulative size of object and its references 对象及其引用的累积大小
}

//格式化输出节点统计对象的信息
func (ns NodeStat) String() string {
   f := "NodeStat{NumLinks: %d, BlockSize: %d, LinksSize: %d, DataSize: %d, CumulativeSize: %d}"
   return fmt.Sprintf(f, ns.NumLinks, ns.BlockSize, ns.LinksSize, ns.DataSize, ns.CumulativeSize)
}
//通过节点创建该节点的链接
// MakeLink creates a link to the given node
func MakeLink(n Node) (*Link, error) {
   s, err := n.Size()
   if err != nil {
      return nil, err
   }

   return &Link{
      Size: s,
      Cid:  n.Cid(),
   }, nil
}
//返回这个链接指向的MDAG节点，用来获取这个链接指向的节点
// GetNode returns the MDAG Node that this link points to
func (l *Link) GetNode(ctx context.Context, serv NodeGetter) (Node, error) {
   return serv.Get(ctx, l.Cid)
}
```





```
package format

import (
   "fmt"
   "sync"

   blocks "gx/ipfs/QmVzK524a2VWLqyvtBeiHKsUAWYgeAk4DBeZoY7vpNPNRx/go-block-format"
)
//函数将块解码成节点
// DecodeBlockFunc functions decode blocks into nodes.
type DecodeBlockFunc func(block blocks.Block) (Node, error)

//块编码器
type BlockDecoder interface {
	//注册块的信息
   Register(codec uint64, decoder DecodeBlockFunc)
   //将块解密 获得节点
   Decode(blocks.Block) (Node, error)
}
type safeBlockDecoder struct {
   // Can be replaced with an RCU if necessary.
   lock     sync.RWMutex
   decoders map[uint64]DecodeBlockFunc
}
//寄存器用已通过的编解码器为所有块注册解码器。
//这将自动替换任何已注册的块解码器。
// Register registers decoder for all blocks with the passed codec.
//
// This will silently replace any existing registered block decoders.
func (d *safeBlockDecoder) Register(codec uint64, decoder DecodeBlockFunc) {
   d.lock.Lock()
   defer d.lock.Unlock()
   d.decoders[codec] = decoder
}

func (d *safeBlockDecoder) Decode(block blocks.Block) (Node, error) {
   // Short-circuit by cast if we already have a Node.
   if node, ok := block.(Node); ok {
      return node, nil
   }

   ty := block.Cid().Type()

   d.lock.RLock()
   decoder, ok := d.decoders[ty]
   d.lock.RUnlock()

   if ok {
      return decoder(block)
   } else {
      // TODO: get the *long* name for this format
      return nil, fmt.Errorf("unrecognized object type: %d", ty)
   }
}

var DefaultBlockDecoder BlockDecoder = &safeBlockDecoder{decoders: make(map[uint64]DecodeBlockFunc)}

// Decode decodes the given block using the default BlockDecoder.
func Decode(block blocks.Block) (Node, error) {
   return DefaultBlockDecoder.Decode(block)
}

// Register registers block decoders with the default BlockDecoder.
func Register(codec uint64, decoder DecodeBlockFunc) {
   DefaultBlockDecoder.Register(codec, decoder)
}
```
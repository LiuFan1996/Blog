# IPFS（三）源码解读之-add





```go
package commands

import (
   "errors"
   "fmt"
   "io"
   "os"
   "strings"
	//块服务提供的接口
   blockservice "github.com/ipfs/go-ipfs/blockservice"
    //核心api
   core "github.com/ipfs/go-ipfs/core"
    //add的一些工具方法和结构
   "github.com/ipfs/go-ipfs/core/coreunix"
    //文件存储接口
   filestore "github.com/ipfs/go-ipfs/filestore"
    //dag服务接口
   dag "github.com/ipfs/go-ipfs/merkledag"
    //提供一个新的线程安全的dag	
   dagtest "github.com/ipfs/go-ipfs/merkledag/test"
    //一个可变IPFS文件系统的内存模型
   mfs "github.com/ipfs/go-ipfs/mfs"
    //文件系统格式
   ft "github.com/ipfs/go-ipfs/unixfs"

    //控制台入口工具包 以下都是工具包
   cmds "gx/ipfs/QmNueRyPRQiV7PUEpnP4GgGLuK1rKQLaRW7sfPvUetYig1/go-ipfs-cmds"
   mh "gx/ipfs/QmPnFwZ2JXKnXgMw8CdBPxn7FWh6LLdjUjxV1fKHuJnkr8/go-multihash"
   pb "gx/ipfs/QmPtj12fdwuAqj9sBSTNUxBNu8kCGNp8b3o8yUzMm5GHpq/pb"
   offline "gx/ipfs/QmS6mo1dPpHdYsVkm27BRZDLxpKBCiJKUH8fHX15XFfMez/go-ipfs-exchange-offline"
   bstore "gx/ipfs/QmadMhXJLHMFjpRmh85XjpmVDkEtQpNYEZNRpWRvYVLrvb/go-ipfs-blockstore"
   cmdkit "gx/ipfs/QmdE4gMduCKCGAcczM2F5ioYDfdeKuPix138wrES1YSr7f/go-ipfs-cmdkit"
   files "gx/ipfs/QmdE4gMduCKCGAcczM2F5ioYDfdeKuPix138wrES1YSr7f/go-ipfs-cmdkit/files"
)

//限制深度 深度达到上限
// ErrDepthLimitExceeded indicates that the max depth has been exceeded.
var ErrDepthLimitExceeded = fmt.Errorf("depth limit exceeded")

//构建命令选项参数常量
const (
   quietOptionName       = "quiet"
   quieterOptionName     = "quieter"
   silentOptionName      = "silent"
   progressOptionName    = "progress"
   trickleOptionName     = "trickle"
   wrapOptionName        = "wrap-with-directory"
   hiddenOptionName      = "hidden"
   onlyHashOptionName    = "only-hash"
   chunkerOptionName     = "chunker"
   pinOptionName         = "pin"
   rawLeavesOptionName   = "raw-leaves"
   noCopyOptionName      = "nocopy"
   fstoreCacheOptionName = "fscache"
   cidVersionOptionName  = "cid-version"
   hashOptionName        = "hash"
)
//管道上限
const adderOutChanSize = 8
//构建一个命令
var AddCmd = &cmds.Command{
    //命令对应的帮助信息
   Helptext: cmdkit.HelpText{
      Tagline: "Add a file or directory to ipfs.",
      ShortDescription: `
Adds contents of <path> to ipfs. Use -r to add directories (recursively).
`,
      LongDescription: `
Adds contents of <path> to ipfs. Use -r to add directories.
Note that directories are added recursively, to form the ipfs
MerkleDAG.

The wrap option, '-w', wraps the file (or files, if using the
recursive option) in a directory. This directory contains only
the files which have been added, and means that the file retains
its filename. For example:

  > ipfs add example.jpg
  added QmbFMke1KXqnYyBBWxB74N4c5SBnJMVAiMNRcGu6x1AwQH example.jpg
  > ipfs add example.jpg -w
  added QmbFMke1KXqnYyBBWxB74N4c5SBnJMVAiMNRcGu6x1AwQH example.jpg
  added QmaG4FuMqEBnQNn3C8XJ5bpW8kLs7zq2ZXgHptJHbKDDVx

You can now refer to the added file in a gateway, like so:

  /ipfs/QmaG4FuMqEBnQNn3C8XJ5bpW8kLs7zq2ZXgHptJHbKDDVx/example.jpg

The chunker option, '-s', specifies the chunking strategy that dictates
how to break files into blocks. Blocks with same content can
be deduplicated. The default is a fixed block size of
256 * 1024 bytes, 'size-262144'. Alternatively, you can use the
rabin chunker for content defined chunking by specifying
rabin-[min]-[avg]-[max] (where min/avg/max refer to the resulting
chunk sizes). Using other chunking strategies will produce
different hashes for the same file.

  > ipfs add --chunker=size-2048 ipfs-logo.svg
  added QmafrLBfzRLV4XSH1XcaMMeaXEUhDJjmtDfsYU95TrWG87 ipfs-logo.svg
  > ipfs add --chunker=rabin-512-1024-2048 ipfs-logo.svg
  added Qmf1hDN65tR55Ubh2RN1FPxr69xq3giVBz1KApsresY8Gn ipfs-logo.svg

You can now check what blocks have been created by:

  > ipfs object links QmafrLBfzRLV4XSH1XcaMMeaXEUhDJjmtDfsYU95TrWG87
  QmY6yj1GsermExDXoosVE3aSPxdMNYr6aKuw3nA8LoWPRS 2059
  Qmf7ZQeSxq2fJVJbCmgTrLLVN9tDR9Wy5k75DxQKuz5Gyt 1195
  > ipfs object links Qmf1hDN65tR55Ubh2RN1FPxr69xq3giVBz1KApsresY8Gn
  QmY6yj1GsermExDXoosVE3aSPxdMNYr6aKuw3nA8LoWPRS 2059
  QmerURi9k4XzKCaaPbsK6BL5pMEjF7PGphjDvkkjDtsVf3 868
  QmQB28iwSriSUSMqG2nXDTLtdPHgWb4rebBrU7Q1j4vxPv 338
`,
   },
//命令对应参数格式
   Arguments: []cmdkit.Argument{
      cmdkit.FileArg("path", true, true, "The path to a file to be added to ipfs.").EnableRecursive().EnableStdin(),
   },
    //命令参数可选项设置 
   Options: []cmdkit.Option{
       //注意所有带有experimental的命令选项都是实验的部分需要在配置文件中启用如果需要使用测试的话
      cmds.OptionRecursivePath, // a builtin option that allows recursive paths (-r, --recursive)
      cmdkit.BoolOption(quietOptionName, "q", "Write minimal output."),
      cmdkit.BoolOption(quieterOptionName, "Q", "Write only final hash."),
      cmdkit.BoolOption(silentOptionName, "Write no output."),
      cmdkit.BoolOption(progressOptionName, "p", "Stream progress data."),
      cmdkit.BoolOption(trickleOptionName, "t", "Use trickle-dag format for dag generation."),
      cmdkit.BoolOption(onlyHashOptionName, "n", "Only chunk and hash - do not write to disk."),
      cmdkit.BoolOption(wrapOptionName, "w", "Wrap files with a directory object."),
      cmdkit.BoolOption(hiddenOptionName, "H", "Include files that are hidden. Only takes effect on recursive add."),
      cmdkit.StringOption(chunkerOptionName, "s", "Chunking algorithm, size-[bytes] or rabin-[min]-[avg]-[max]").WithDefault("size-262144"),
      cmdkit.BoolOption(pinOptionName, "Pin this object when adding.").WithDefault(true),
      cmdkit.BoolOption(rawLeavesOptionName, "Use raw blocks for leaf nodes. (experimental)"),
      cmdkit.BoolOption(noCopyOptionName, "Add the file using filestore. Implies raw-leaves. (experimental)"),
      cmdkit.BoolOption(fstoreCacheOptionName, "Check the filestore for pre-existing blocks. (experimental)"),
      cmdkit.IntOption(cidVersionOptionName, "CID version. Defaults to 0 unless an option that depends on CIDv1 is passed. (experimental)"),
      cmdkit.StringOption(hashOptionName, "Hash function to use. Implies CIDv1 if not sha2-256. (experimental)").WithDefault("sha2-256"),
   },
    //设置命令默认选项的默认值，节点启动时运行
   PreRun: func(req *cmds.Request, env cmds.Environment) error {
      quiet, _ := req.Options[quietOptionName].(bool)
      quieter, _ := req.Options[quieterOptionName].(bool)
      quiet = quiet || quieter

      silent, _ := req.Options[silentOptionName].(bool)

      if quiet || silent {
         return nil
      }

      // ipfs cli progress bar defaults to true unless quiet or silent is used
      _, found := req.Options[progressOptionName].(bool)
      if !found {
         req.Options[progressOptionName] = true
      }

      return nil
   },
    //在控制台命令时调用run
   Run: func(req *cmds.Request, res cmds.ResponseEmitter, env cmds.Environment) {
       //获取IPFSNode的配置信息
      n, err := GetNode(env)
      if err != nil {
         res.SetError(err, cmdkit.ErrNormal)
         return
      }
		//获取IPFS全局配置文件配置信息
      cfg, err := n.Repo.Config()
      if err != nil {
         res.SetError(err, cmdkit.ErrNormal)
         return
      }
      // check if repo will exceed storage limit if added
      // TODO: this doesn't handle the case if the hashed file is already in blocks (deduplicated)
      // TODO: conditional GC is disabled due to it is somehow not possible to pass the size to the daemon
      //if err := corerepo.ConditionalGC(req.Context(), n, uint64(size)); err != nil {
      // res.SetError(err, cmdkit.ErrNormal)
      // return
      //}
	
       //将所有的命令参数对应的值强转bool 以用来验证该命令参数有没有被使用
      progress, _ := req.Options[progressOptionName].(bool)
      trickle, _ := req.Options[trickleOptionName].(bool)
      wrap, _ := req.Options[wrapOptionName].(bool)
      hash, _ := req.Options[onlyHashOptionName].(bool)
      hidden, _ := req.Options[hiddenOptionName].(bool)
      silent, _ := req.Options[silentOptionName].(bool)
      chunker, _ := req.Options[chunkerOptionName].(string)
      dopin, _ := req.Options[pinOptionName].(bool)
      rawblks, rbset := req.Options[rawLeavesOptionName].(bool)
      nocopy, _ := req.Options[noCopyOptionName].(bool)
      fscache, _ := req.Options[fstoreCacheOptionName].(bool)
      cidVer, cidVerSet := req.Options[cidVersionOptionName].(int)
      hashFunStr, _ := req.Options[hashOptionName].(string)

      // The arguments are subject to the following constraints.
      //
      // nocopy -> filestoreEnabled
      // nocopy -> rawblocks
      // (hash != sha2-256) -> cidv1

      // NOTE: 'rawblocks -> cidv1' is missing. Legacy reasons.

      // nocopy -> filestoreEnabled
       //实验方法，具体可以自行实验
      if nocopy && !cfg.Experimental.FilestoreEnabled {
         res.SetError(filestore.ErrFilestoreNotEnabled, cmdkit.ErrClient)
         return
      }
		//实验方法，具体可以自行实验
      // nocopy -> rawblocks
      if nocopy && !rawblks {
         // fixed?
         if rbset {
            res.SetError(
               fmt.Errorf("nocopy option requires '--raw-leaves' to be enabled as well"),
               cmdkit.ErrNormal,
            )
            return
         }
         // No, satisfy mandatory constraint.
         rawblks = true
      }
	//实验方法，具体可以自行实验	
      // (hash != "sha2-256") -> CIDv1
      if hashFunStr != "sha2-256" && cidVer == 0 {
         if cidVerSet {
            res.SetError(
               errors.New("CIDv0 only supports sha2-256"),
               cmdkit.ErrClient,
            )
            return
         }
         cidVer = 1
      }
	//实验方法，具体可以自行实验
      // cidV1 -> raw blocks (by default)
      if cidVer > 0 && !rbset {
         rawblks = true
      }
	//实验方法，具体可以自行实验
      prefix, err := dag.PrefixForCidVersion(cidVer)
      if err != nil {
         res.SetError(err, cmdkit.ErrNormal)
         return
      }
	
      hashFunCode, ok := mh.Names[strings.ToLower(hashFunStr)]
      if !ok {
         res.SetError(fmt.Errorf("unrecognized hash function: %s", strings.ToLower(hashFunStr)), cmdkit.ErrNormal)
         return
      }

      prefix.MhType = hashFunCode
      prefix.MhLength = -1
		//如果使用 -n 命令参数 只写入块hash，不写入磁盘
      if hash {
         nilnode, err := core.NewNode(n.Context(), &core.BuildCfg{
            //TODO: need this to be true or all files
            // hashed will be stored in memory!
            NilRepo: true,
         })
         if err != nil {
            res.SetError(err, cmdkit.ErrNormal)
            return
         }
         n = nilnode
      }
		//一个可以回收的块存储 
      addblockstore := n.Blockstore
       //如果true 就构建一个新的可以回收的块
      if !(fscache || nocopy) {
         addblockstore = bstore.NewGCBlockstore(n.BaseBlocks, n.GCLocker)
      }
		//基本上不会被执行，可能是版本跟新代码没有删除干净
      exch := n.Exchange
      local, _ := req.Options["local"].(bool)
      if local {
         exch = offline.Exchange(addblockstore)
      }
	 //通过块服务构建一个新的块启用块的交换策略
      bserv := blockservice.New(addblockstore, exch) // hash security 001
     //将块交给dag服务管理
       dserv := dag.NewDAGService(bserv)
		//新建输出管道 设置长度
      outChan := make(chan interface{}, adderOutChanSize)
		//返回于文件添加操作的新文件对象
      fileAdder, err := coreunix.NewAdder(req.Context, n.Pinning, n.Blockstore, dserv)
      if err != nil {
         res.SetError(err, cmdkit.ErrNormal)
         return
      }
		//为文件对象设置属性
      fileAdder.Out = outChan
      fileAdder.Chunker = chunker
      fileAdder.Progress = progress
      fileAdder.Hidden = hidden
      fileAdder.Trickle = trickle
      fileAdder.Wrap = wrap
      fileAdder.Pin = dopin
      fileAdder.Silent = silent
      fileAdder.RawLeaves = rawblks
      fileAdder.NoCopy = nocopy
      fileAdder.Prefix = &prefix
		//如果使用 -n 命令参数 只写入块hash，不写入磁盘
      if hash {
          //获取一个新的线程安全的dag
         md := dagtest.Mock()
         emptyDirNode := ft.EmptyDirNode()
         // Use the same prefix for the "empty" MFS root as for the file adder.
         emptyDirNode.Prefix = *fileAdder.Prefix
         mr, err := mfs.NewRoot(req.Context, md, emptyDirNode, nil)
         if err != nil {
            res.SetError(err, cmdkit.ErrNormal)
            return
         }

         fileAdder.SetMfsRoot(mr)
      }
		//构建文件上传io
      addAllAndPin := func(f files.File) error {
         // Iterate over each top-level file and add individually. Otherwise the
         // single files.File f is treated as a directory, affecting hidden file
         // semantics.
          //每次读取一个文件保存到新的文件对象fileAdder中
         for {
            file, err := f.NextFile()
            if err == io.EOF {
               // Finished the list of files.
               break
            } else if err != nil {
               return err
            }
            if err := fileAdder.AddFile(file); err != nil {
               return err
            }
         }
		
         // copy intermediary nodes from editor to our actual dagservice
          // Finalize方法刷新mfs根目录并返回mfs根节点。
         _, err := fileAdder.Finalize()
         if err != nil {
            return err
         }

         if hash {
            return nil
         }
		//递归新的我那件对象和根节点
		//将pin节点状态写入后台数据存储。
         return fileAdder.PinRoot()
      }

      errCh := make(chan error)
       
       //开启协程进行文件上传
      go func() {
          //一个错误变量
         var err error
          //defer一个管道接收err变量 存放文件传输过程中出现的错误
         defer func() { errCh <- err }()
         defer close(outChan)
          //传输文件并返回错误信息
         err = addAllAndPin(req.Files)
      }()
		//关闭链接
      defer res.Close()
		//res 错误
      err = res.Emit(outChan)
      if err != nil {
         log.Error(err)
         return
      }
       //传输错误
      err = < -errCh
      if err != nil {
         res.SetError(err, cmdkit.ErrNormal)
      }
   },
    //返回执行结果到命令行
   PostRun: cmds.PostRunMap{
       //实习接口方法
      cmds.CLI: func(req *cmds.Request, re cmds.ResponseEmitter) cmds.ResponseEmitter {
          //新建一个输出通道标准格式
         reNext, res := cmds.NewChanResponsePair(req)
          //add 命令行返回值 文件hash信息 存储管道
         outChan := make(chan interface{})
		  //add 命令行方悔之 文件大小    存储管道
         sizeChan := make(chan int64, 1)
		//通过文件对象获取文件大小对象
         sizeFile, ok := req.Files.(files.SizeFile)
          //如果获取文件对象成功
         if ok {
            // Could be slow.
            go func() {
                //通过文件对象获取大小
               size, err := sizeFile.Size()
               if err != nil {
                  log.Warningf("error getting files size: %s", err)
                  // see comment above
                  return
               }
				//将文件大小存到文件大小管道中
               sizeChan <- size
            }()
         } else {
             //不能获得上传文件的大小
            // we don't need to error, the progress bar just
            // won't know how big the files are
            log.Warning("cannot determine size of input file")
         }
			//进度条
         progressBar := func(wait chan struct{}) {
            defer close(wait)

            quiet, _ := req.Options[quietOptionName].(bool)
            quieter, _ := req.Options[quieterOptionName].(bool)
            quiet = quiet || quieter

            progress, _ := req.Options[progressOptionName].(bool)
			
            var bar *pb.ProgressBar
            if progress {
               bar = pb.New64(0).SetUnits(pb.U_BYTES)
               bar.ManualUpdate = true
               bar.ShowTimeLeft = false
               bar.ShowPercent = false
               bar.Output = os.Stderr
               bar.Start()
            }

            lastFile := ""
            lastHash := ""
            var totalProgress, prevFiles, lastBytes int64

         LOOP:
            for {
               select {
               case out, ok := <-outChan:
                  if !ok {
                     if quieter {
                        fmt.Fprintln(os.Stdout, lastHash)
                     }

                     break LOOP
                  }
                  output := out.(*coreunix.AddedObject)
                  if len(output.Hash) > 0 {
                     lastHash = output.Hash
                     if quieter {
                        continue
                     }

                     if progress {
                        // clear progress bar line before we print "added x" output
                        fmt.Fprintf(os.Stderr, "\033[2K\r")
                     }
                     if quiet {
                        fmt.Fprintf(os.Stdout, "%s\n", output.Hash)
                     } else {
                        fmt.Fprintf(os.Stdout, "added %s %s\n", output.Hash, output.Name)
                     }

                  } else {
                     if !progress {
                        continue
                     }

                     if len(lastFile) == 0 {
                        lastFile = output.Name
                     }
                     if output.Name != lastFile || output.Bytes < lastBytes {
                        prevFiles += lastBytes
                        lastFile = output.Name
                     }
                     lastBytes = output.Bytes
                     delta := prevFiles + lastBytes - totalProgress
                     totalProgress = bar.Add64(delta)
                  }

                  if progress {
                     bar.Update()
                  }
               case size := <-sizeChan:
                  if progress {
                     bar.Total = size
                     bar.ShowPercent = true
                     bar.ShowBar = true
                     bar.ShowTimeLeft = true
                  }
               case <-req.Context.Done():
                  // don't set or print error here, that happens in the goroutine below
                  return
               }
            }
         }
			//控制文件上传时的制度条显示
         go func() {
            // defer order important! First close outChan, then wait for output to finish, then close re
            defer re.Close()

            if e := res.Error(); e != nil {
               defer close(outChan)
               re.SetError(e.Message, e.Code)
               return
            }

            wait := make(chan struct{})
            go progressBar(wait)

            defer func() { <-wait }()
            defer close(outChan)

            for {
               v, err := res.Next()
               if !cmds.HandleError(err, res, re) {
                  break
               }

               select {
               case outChan <- v:
               case <-req.Context.Done():
                  re.SetError(req.Context.Err(), cmdkit.ErrNormal)
                  return
               }
            }
         }()

         return reNext
      },
   },
    //添加一个object对象
   Type: coreunix.AddedObject{},
}
```
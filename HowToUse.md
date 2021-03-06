# How To Use

## 使用Pipeline设计模式实现打包流程

SmartBundleBuilder使用了[Pipeline](https://medium.com/@aaronweatherall/the-pipeline-pattern-for-fun-and-profit-9b5f43a98130)设计模式，
pipeline模式优点

* 每个处理步骤更独立

* 更有利于重用

* 可以根据需要实现自己的pipeline流程

举例：比如音乐文件不需要进行依赖分析的，可以删除前4步。


### 内置的标准的流程如图：

![GitHub](https://github.com/terrynoya/SmartBundleBuilder/raw/master/doc/pipeline.jpeg)


### Pipeline结构

#### Pipeline
Pipeline是一个list容器，list中存放ValeBase节点

#### ValeBase

ValeBase是pipeline中的节点，用于处理具体问题

派生子类通过实现ValeBase.Excute(Payload)方法来满足具体的需求

例如SmartBundleBuilder中的FileDependcyAnylize类是ValvBase的子类之一，用来生成文件依赖关系有向图

你也可以按照自己的需求实现ValveBase子类处理payload

### **ValveBase完成处理后调用Complete()，如果失败，调用Error()**


##### FileDependcyAnylize代码举例

---
```C#
/// <summary>
/// 资源依赖关系分析
/// </summary>
namespace SmartBundle
{
    public class FileDependcyAnylize : ValveBase
    {
        //通过覆盖这个方法实现具体功能
        public override void Excute(Payload payload)
        {
            BundlePayload context = (BundlePayload) payload;
            MAssetFileDependencyGraphGenerator fileGraphGen =
                new MAssetFileDependencyGraphGenerator(context.FileDependcyGraph);
            string[] filePathList = context.Files;
            for (int i = 0; i < filePathList.Length; i++)
            {
                string filePath = filePathList[i];
                fileGraphGen.AddFileFromPath(filePath);
            }
            MDirectedGraph graph = fileGraphGen.Graph;
            //删除多余依赖关系
            TransitiveReduction.Reduce(graph);

            this.Complete();
        }
    }
}
```

#### Payload

Payload是pipeline中需要处理的上下文，SmartBundleBuilder中的BundlePayload类是payload的子类，用来存放pipeline处理后产生的数据，或者给ValveBase提供参数

BundlePayload如下

```C#
public class BundlePayload: Payload
{
    //需要处理的文件列表
    public string[] Files;
    //FileDependcyAnylize执行后，产生的文件依赖关系有向图
    public MDirectedGraph FileDependcyGraph;
    public MDirectedGraph BundleDependcyGraph;

    public IBuildOptionParser OptionParser;
    public AssetBundleBuildConfigVO BulidConfig;
    //Assetbundle输出目录
    public string OutputPath;
    public string OuputReletivePath;
    //Assetbundle输出平台
    public BuildEnums.BuildPlatform Platform;
    //Assetbundle扩展名
    public string BundleExtension;

    public BundlePayload()
    {
        this.FileDependcyGraph = new MDirectedGraph();
        this.BundleDependcyGraph = new MDirectedGraph();
    }
}
```

#### Process()

使用pipeline.Process()方法执行pipeline，pipeline会按照顺序执行ValveBase，如果ValveBase返回了Error，则中断pipeline


## 打包pipeline的标准流程代码实现

---
```C#
/// <summary>
/// 标准处理管道
/// </summary>
/// <param name="files"></param>
/// <param name="outputPath"></param>
/// <returns></returns>
public static Pipeline CreateStandardPipeline()
{
    Pipeline pipe = new Pipeline();

    pipe.Pipe(new FileDependcyAnylize())
        .Pipe(new LogFileDependcyGraph())
        .Pipe(new MergeFileToPackage())
        .Pipe(new PackageGraphCircleCheck())
        .Pipe(new ClearBundleNames())
        .Pipe(new SetBundleNames())
        .Pipe(new BuildBundle())
        .Pipe(new GenerateHintFile())
        .Pipe(new RenameManifestBinaryFile())
        .Pipe(new RemoveAllManifestTextFiles());

    return pipe;
}
```

## 通过反射配置pipeline

SmartBundleBuilder支持通过反射构建pipeline，你可以使用Rules.txt配置文件，设定需要打包的参数和流程


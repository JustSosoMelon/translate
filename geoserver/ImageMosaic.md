##  Using the ImageMosaic plugin
### 介绍
这个教程描述了使用ImageMosaic创建一个新图层的过程。这个插件是geotools提供的，允许根据一系列地图参考栅格文件创建一个拼接图。这个插件可以识别geotiffs，以及带world文件描述的其他栅格文件，如pgw和png文件、jgw文件和jpg文件等等。另外，如果安装了imageio-ext gdal扩展，就可以支持gdal支持所有格式，如MrSid，ECW，JPEG2000等等，如何安装gdal及更多相关信息参考[gdal图片格式](http://docs.geoserver.org/2.8.1/user/data/raster/gdal.html#data-gdal)
JAI文档给出了一个Mosaic的描述：
“Mosaic”操作根据两个或者多个源文件创建一个拼接图，这种操作可以帮助将一系列地理空间精确的图片聚集成一个连续图，这可以用来制作大量图片的合成图，如全景图。

简单来说，ImageMosaic插件用于将一系列相似的栅格数据组合在一起，从现在起，我们称这些栅格数据为*granules*，这个插件当然也有一些限制：
 1. 所有granules必须使用相同的坐标系
 2. 所有granules必须共享相同的ColorModel和SampleModel，这是底层的JAI Mosaic操作的限制：意味着granules必须共享相同的像素布局和光栅解释，这个限制很难克服，但是一些扩展可以做到。注意，在colormapped granules的情况下，如果多个granules共享相同的colormap，代码将尽最大努力保留它，不会在内存里扩展相同的colormap，这个动作也可以通过配置文件中的一个参数控制，见下一section。
 3. 所有granules必须共享相同的空间分辨率和一组概述（如果概述不共享，概述将会被抛弃）。
 
 ### Granule Index
为了用这个插件配置一个新的图层存储和一个新的图层，必须先生成一个索引文件，关联每个granule和它的边界，当前仅支持一个shape文件作为一个索引，尽管扩展它去使用其他方式持久化索引是可能的。
更具体一点，下面的文件组成了mosaic的配置：
1. 一个shapefile，包含了每个栅格文件的封闭多边形，这个shapefile需要有一个属性，其值为mosaic granule的路径，这个路径可以是相对shapefile的路径或绝对路径，这个属性的名字默认是“location”，也可以被配置成其他的名字（后续会讨论）。
2. 一个projection file（.prj），描述上面提到的shapefile。
3. 一个config file（.properties），这个文件包含的属性有：x和y方向的网格size，mosaic图层的个数，等等，我们将在下一部分描述这个文件。
当配置mosaic存储指向一个包含图片的目录时，通常Geoserver会自动创建以上文件，但理解他们对理解mosaic如何工作和定位问题都是很重要的。

### 配置文件
mosaic配置文件用于存储配置参数，控制ImageMosaic产检，它作为mosaic创建的一部分，通常不需要人工编辑，下面的表格描述了配置文件中的多个元素：
| 参数 |是否强制  | 描述 |
|--|--|--|--|
| Evelope2D | Y | Contains the envelope for this mosaic formatted as LLCx,LLXy URCx,URCy (notice the space between the coordinates of the Lower Left Corner and the coordinates of the Upper Right Corner). An example is _Envelope2D=432500.25,81999.75 439250.25,84999.75_ |  |
| LevelsNum | Y | Represents the number of reduced resolution layers that we currently have for the granules of this mosaic. |  |
| Levels | Y | Represents the number of reduced resolution layers that we currently have for the granules of this mosaic. |  |
| Name | Y | Name of the mosaic |  |
| ExpandToRGB | N | Applies to colormapped granules. Asks the internal mosaic engine to expand the colormapped granules to RGB prior to mosaicking them. This is needed whenever the the granules do not share the same color map hence a straight composition that would retain such a color map cannot be performed. |  |
| AbsolutePath | Y |  It controls whether or not the path stored inside the “location” attribute represents an absolute path or a path relative to the location of the shapefile index. Notice that a relative index ensure much more portability of the mosaic itself. Default value for this parameter is False, which means relative paths. |  |
| LocationAttribute | N | The name of the attribute path in the shapefile index. Default value is  location |  |

### 创建Granules索引和配置文件
重构版本后的ImageMosaic插件可以在线生成shapefile索引和mosaic配置文件，不需要依赖gdal或其他类似的工具。
如果你有一个目录数，包含所有你想要组成mosaic的granules，你需要做的就是将geoserver data store指向这个目录，他会自动检查目录树中从提供的输入开始的所有文件，创建合理的辅助文件。

### 配置一个图层
这个过程和创建一个feature type类似，详细请参考[相关部分](http://docs.geoserver.org/2.8.1/user/tutorials/image_mosaic_plugin/imagemosaic.html#configuring-a-coverage-in-geoserver)

> 注意：如果创建的图层展示时都是黑色，可能是geoserver在当前的index中还没有找到任何可接受的granules。也可能是shapefile索引为空（配置提供的目录中未找到任何granules），或者shapefile中的路径属性（granules的路径）不正确（可能是使用了绝对路径，但是文件移动到了其他目录）。如果shapefile索引中路径是错误的，可以使用openoffice打开dbf文件修正它。另一个选择是删除index并且让geoserver重新创建它。

### 调整ImageMosaic图层配置
图层编辑页面提供了一组控制参数，用于进一步调整或控制mosaic创建的过程，参数如下：
| 参数 | 描述 |
|--|--|
| MaxAllowedTiles | 一个请求同时允许加载的最大tile数目，对于大的mosaic，这个参数应该被恰当的设置，防止服务器同时加载太多granules，从而耗尽资源 |
| BackgroundValues | 设置mosaic的背景，基于mosaic的特性，为no data区域设置一个值是明智的，通常-9999，这个值在所有mosaic band上生效 |
| Filter | 设置默认的mosaic过滤器，应该是一个有效的ECQL查询，如果没有设置cql_filter,该查询 valid query，类似select 1，将会被用于默认，而不是Filter.INCLUDE,如果cql_filter在请求中指定了，则会覆盖默认值 |

> 注意：不要使用filter去改变时间或者海拔维度。他会被添加为And条件，对time来说是与CURRENT做And，对elevation来说是与LOWER做与。
> **OutputTransparentColor**
> 为创建mosaic设置透明color，参见以下的[example](http://docs.geoserver.org/2.8.1/user/tutorials/image_mosaic_plugin/imagemosaic.html#tweaking-an-imagemosaic-coveragestore)：
> 为了去控制granules之间的重叠过程，在mosaicking之前为granules设置透明color。当geoserver组合granules满足用户请求，一些granule会覆盖其他granule，因此设置这个参数避免了granules之间no data区域的重叠，见如下的[example](http://docs.geoserver.org/2.8.1/user/tutorials/image_mosaic_plugin/imagemosaic.html#tweaking-an-imagemosaic-coveragestore)

### 其他参数
| 参数 | 描述 |
|--|--|
| AllowMultithreading | tr.ue表示允许tiles的(多线程)（加载） |
| USE_JAI_IMAGEREAD | 控制底层读取granules的机制，true表示geoserver会使用JAI图片读取操作和延迟加载机制，false表示使用Java ImageIO读取图片，即时加载 |
| SUGGESTED_TILE_SIZE | 控制输入granules和输出mosaic的tile size，由逗号分隔的两个正整数组成，如512，512 |

> 注意：延迟加载消耗更少的内存，因为它使用流加载，每次只有process需要的数据才会加载，但是，另一方面，可能造成的问题是较重的负载，因为它长时间保持granules file的打开以支持延迟加载。
> 注意：即时加载消耗更多的内存，因为它加载整个请求的mosaic到内存，但是另一方面，它的性能更好，不会遇到延迟加载可能遇到的系统too many files open的错误。

### 配置例子
[参考](http://docs.geoserver.org/2.8.1/user/tutorials/image_mosaic_plugin/imagemosaic.html#configuration-examples)

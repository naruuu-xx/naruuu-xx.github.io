---
title: "uniapp图片上传服务器并回显"
date: 2022-10-23T20:36:22+08:00
draft: false
---

之前写了一些关于小程序的项目。
这里总结一下 uniapp 下使用 uview 以及 springboot 如何实现小程序图片上传。

首先，可以确定的是，uview 本身是存在有图片上传组件的，因此我们所需要做的便是编写图片上传的 api 。

这里给出图片上传的后端代码。可以直接拿去用。
```java
@RestController  
@RequestMapping("/upload")  
public class UploadController {  
    private static final Logger logger = LoggerFactory.getLogger(UploadController.class);  
  
    private SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyyMMdd");  
  
  
  
    @Value("${file-save-path}")  
    private String fileSavePath;  
  
    @RequestMapping("/picture")  
    public String uploadPicture(HttpServletRequest request, HttpServletResponse response) throws Exception {  
        String filePath = "";  
        request.setCharacterEncoding("utf-8"); //设置编码  
        //String realPath = request.getSession().getServletContext().getRealPath("/uploadFile/");  
        String directory =simpleDateFormat.format(new Date());  
        //System.out.println(realPath);  
        File dir = new File(fileSavePath+directory);  
        System.out.println(dir.getPath());  
        //文件目录不存在，就创建一个  
        if (!dir.isDirectory()) {  
            dir.mkdirs();  
        }  
        try {  
            StandardMultipartHttpServletRequest req = (StandardMultipartHttpServletRequest) request;  
            //获取formdata的值  
            Iterator<String> iterator = req.getFileNames();  
  
            while (iterator.hasNext()) {  
                MultipartFile file = req.getFile(iterator.next());  
                String fileName = file.getOriginalFilename();  
                //真正写到磁盘上  
  
                String suffix = file.getOriginalFilename().substring(file.getOriginalFilename().lastIndexOf("."));  
                String newFileName= UUID.randomUUID().toString().replaceAll("-", "")+suffix;  
                File file1 = new File(fileSavePath+directory+"/" + newFileName);  
                file.transferTo(file1);  
                System.out.println(file1.getPath());  
                filePath = request.getScheme() + "://" +  
                        request.getServerName() + ":"  
                        + request.getServerPort()  
                        +"/images/" +directory+"/"+ newFileName;  
  
                System.out.println("访问图片路径:" + filePath);  
            }  
        } catch (Exception e) {  
            logger.error("", e);  
        }  
        return filePath;  
    }  
  
  
}
```


然后出现的疑问便是：
既然是上传图片，至少要知道图片要上传到哪里吧？

这里我们就需要进行路径配置啦。因为涉及到页面回显，与此同时我也做了映射。
当访问/images/时，可以显示到上传后的图片。

application.yml 中进行上传路径的配置:

```yml
file-save-path: #这里配置你需要上传的路径


```


想要通过url访问到上传的路径的话，我们就需要进行路径映射的配置了。这里给出代码：
```java
  
@Configuration  
public class WebConfig implements WebMvcConfigurer {  
  
    @Value("${file-save-path}")  
    private String fileSavePath;  
  
    @Override  
    public void addResourceHandlers(ResourceHandlerRegistry registry) {  
        /**  
         * 配置资源映射  
         * 意思是：如果访问的资源路径是以“/images/”开头的，  
         * 就给我映射到本机的“E:/images/”这个文件夹内，去找你要的资源  
         * 注意：E:/images/ 后面的 “/”一定要带上  
         */  
        registry.addResourceHandler("/images/**").addResourceLocations("file:"+fileSavePath);  
  
    }  
}
```


小程序端代码显示：

```js
<template>
	<view class="content">
		<u-upload :fileList="fileList1" @afterRead="afterRead" @delete="deletePic"
			name="1" multiple :maxCount="1" width="200" height="200"></u-upload>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				title: 'Hello',
				fileList1: [],
			}
		},
		onLoad() {

		},
		methods: {
			deletePic(event) {
				this[`fileList${event.name}`].splice(event.index, 1)
			},
			// 新增图片
			async afterRead(event) {
				// 当设置 mutiple 为 true 时, file 为数组格式，否则为对象格式
				let lists = [].concat(event.file)
				console.log(event)
				console.log(this[`fileList${event.name}`])
				let fileListLen = this[`fileList${event.name}`].length
				lists.map((item) => {
					this[`fileList${event.name}`].push({
						...item,
						status: 'uploading',
						message: '上传中'
					})
				})
				for (let i = 0; i < lists.length; i++) {
					const result = await this.uploadFilePromise(lists[i].url)
					let item = this[`fileList${event.name}`][fileListLen]
					this[`fileList${event.name}`].splice(fileListLen, 1, Object.assign(item, {
						status: 'success',
						message: '',
						url: result
					}))
					fileListLen++
				}
			},
			uploadFilePromise(url) {
				return new Promise((resolve, reject) => {
					console.log(this.fileList1[0].url);
					let a = uni.uploadFile({
						url: 'http://localhost:8080/upload/picture', // 仅为示例，非真实的接口地址
						filePath: this.fileList1[0].url,
						name: 'file',
						formData: {
							user: 'test'
						},
						success: (res) => {
							getApp().globalData.imageUrl=res.data;
							//this.model1.imageUrl = res.data;
							setTimeout(() => {
								resolve(res.data.data)
							}, 1000)
						}
					});
				})
			},
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200rpx;
		width: 200rpx;
		margin-top: 200rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}

	.text-area {
		display: flex;
		justify-content: center;
	}

	.title {
		font-size: 36rpx;
		color: #8f8f94;
	}
</style>

```

效果展示：

小程序端：
![](https://s2.loli.net/2022/09/18/d2zZQ1UnHfPB9xJ.png)

web端 网页回显：

这里可能会出现一个问题便是：当上传图片过大时，会上传失败。这是因为springboot 对上传文件大小做了限制。

65.5 Handling Multipart File Uploads
Spring Boot embraces the Servlet 3 javax.servlet.http.Part API to support uploading files. By default Spring Boot configures Spring MVC with a maximum file of 1Mb per file and a maximum of 10Mb of file data in a single request. You may override these values, as well as the location to which intermediate data is stored (e.g., to the /tmp directory) and the threshold past which data is flushed to disk by using the properties exposed in the MultipartProperties class. If you want to specify that files be unlimited, for example, set the multipart.maxFileSize property to -1.The multipart support is helpful when you want to receive multipart encoded file data as a @RequestParam-annotated parameter of type MultipartFile in a Spring MVC controller handler method.

文档说明表示，每个文件的配置最大为1Mb，单次请求的文件的总数不能大于10Mb。要更改这个默认值需要在配置文件（如application.properties）中加入两个配置

需要设置以下两个参数

multipart.maxFileSize
multipart.maxRequestSize

在 application.properties 中进行配置：

```xml
spring.servlet.multipart.max-file-size=100MB
spring.servlet.multipart.max-request-size=1000MB
```

这样，较大的图片就可以上传成功啦。
web端回显效果：

![](https://s2.loli.net/2022/09/18/p3UxiG96olWFdHh.png)
---
sidebar_position: 6
---

# 6. 上传下载

:::tip 上传

记得先导入接口方法 getAPI、xxxApi，然后将文件的值 param.file 传递给接口，可以带参数。上传组件采用 el-upload。
:::

页面布局1：

```csharp
  <el-upload class="inline-block" action="" :show-file-list="false" accept=".xls,.xlsx,.csv" :http-request="(param: any) => uploadxxx(param)">
    <el-button icon="ele-Upload" type="primary" :loading="state.uploading"> {{ state.uploading ? '正在上传' : '上传xxx' }} </el-button>
  </el-upload>
```

页面布局2：也可以参考系统自带的文件上传页面，弹出框模式进行文件上传：

```csharp
  <el-dialog v-model="state.dialogUploadVisible" :lock-scroll="false" draggable width="400px">
   <template #header>
    <div style="color: #fff">
     <el-icon size="16" style="margin-right: 3px; display: inline; vertical-align: middle"> <ele-UploadFilled /> </el-icon>
     <span> 上传文件 </span>
    </div>
   </template>
   <div>
    <el-upload ref="uploadRef" drag :auto-upload="false" :limit="1" :file-list="state.fileList" action="" :on-change="handleChange" accept=".jpg,.png,.bmp,.gif,.txt,.pdf,.xlsx,.docx">
     <el-icon class="el-icon--upload">
      <ele-UploadFilled />
     </el-icon>
     <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
     <template #tip>
      <div class="el-upload__tip">请上传大小不超过 10MB 的文件</div>
     </template>
    </el-upload>
   </div>
   <template #footer>
    <span class="dialog-footer">
     <el-button @click="state.dialogUploadVisible = false">取消</el-button>
     <el-button type="primary" @click="uploadFile">确定</el-button>
    </span>
   </template>
  </el-dialog>
import { getAPI } from '/@/utils/axios-utils';
import { xxxApi } from '/@/api-services/api';

```

```csharp
// 上传
const uploadxxx = async (param: any) => {
 state.loading = true;
 try {
  // // FormData对象，添加参数只能通过append('key', value)的形式添加
  // let formData = new FormData();
  // formData.append('file', param.file);

  state.uploading = true;
  var res = await getAPI(xxxApi).apixxxPostForm(param.file);
 } catch (error) {
  /* empty */
 } finally {
  state.uploading = false;
 }
 state.loading = false;
};
```

:::tip 下载

记得先导入接口方法 getAPI、xxxApi，切记参数为 responseType: 'blob'，否则下载不成功。
:::

```csharp
import { downloadByData, getFileName } from '/@/utils/download';

import { getAPI } from '/@/utils/axios-utils';
import { xxxApi } from '/@/api-services/api';

// 下载
const exportTemp = async () => {
 state.loading = true;
 var res = await getAPI(xxxApi).apixxxPost({ responseType: 'blob' });
 state.loading = false;

 var fileName = getFileName(res.headers);
 downloadByData(res.data as any, fileName);
};
```

**文件预览请参考**  [文件预览](https://gitee.com/zuohuaijun/Admin.NET/blob/next/Web/src/views/system/file/index.vue#L247)

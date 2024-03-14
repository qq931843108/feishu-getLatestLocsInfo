<template>
  <el-form ref="form" class="form" label-position="top">
    
    <el-alert style="margin-bottom: 20px;background-color: #e1eaff;color: #606266;"  :title="$t('alerts.selectNumberField')" type="info" />
    <el-form-item :label="$t('labels.api')" size="large" required>
      <span>{{ $t('labels.apiConfig.address') }}</span><el-input v-model="interfaceAddr" :placeholder="$t('placeholder.apiAddress')"></el-input>
      <span>{{ $t('labels.apiConfig.appcode') }}</span><el-input type="password" v-model="appcode" :placeholder="$t('placeholder.apiAppcode')"></el-input>
      <el-link style="color: #3e75f5;" type="primary" href="https://jfsq6znqku.feishu.cn/wiki/Pr0kwAAn9iQMIakrZK1coFIynhf?from=from_copylink"
      target="_blank">{{ $t('apiDescDocument') }}</el-link>
    </el-form-item>
    <el-form-item :label="$t('labels.number')" size="large" required>
      <el-select v-model="numberFieldId" :placeholder="$t('placeholder.number')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    
    <el-form-item :label="$t('labels.latestTime')" size="large" required>
      <el-select v-model="latestTimeFieldId" :placeholder="$t('placeholder.latestTime')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('labels.latestStatus')" size="large" required>
      <el-select v-model="latestStatusFieldId" :placeholder="$t('placeholder.latestStatus')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('labels.diff')" size="large" required>
      <el-select v-model="diffFieldId" :placeholder="$t('placeholder.diff')" style="width: 100%">
        <el-option v-for="meta in fieldListSeView" :key="meta.id" :label="meta.name" :value="meta.id" />
      </el-select>
    </el-form-item>
    <el-button @click="debouncedFunction" color="#3370ff" type="primary" plain size="large">{{ $t('submit') }}</el-button>
  </el-form>
</template>

<script setup>
import { bitable } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted } from 'vue';
import axios from 'axios';



// -- 数据区域
const { t } = useI18n();
const fieldListSeView = ref([])
const interfaceAddr = ref('https://wuliu.market.alicloudapi.com/kdi')
// const appcode = ref('69d22a9c7e8b414b8a09b115fbb39ee8')
const appcode = ref('')

const diffInfo = ref('')

const numberFieldId = ref('')
const latestTimeFieldId = ref('')
const latestStatusFieldId = ref('')
const diffFieldId = ref('')

// -- 核心算法区域
// --001== 表单最终按钮触发，数据写回
const writeData = async () => {
  // 非空判断
  if (!(interfaceAddr.value && appcode.value && numberFieldId.value && latestTimeFieldId.value && latestStatusFieldId.value && diffFieldId.value)) {
    await bitable.ui.showToast({
      toastType: 'warning',
      message: '缺少必填信息'
    })
  }

  // 缓存处理
  localStorage.setItem('numberFieldId', numberFieldId.value)   // string 类型
  localStorage.setItem('latestTimeFieldId', latestTimeFieldId.value)   // string 类型
  localStorage.setItem('latestStatusFieldId', latestStatusFieldId.value)   // string 类型
  localStorage.setItem('diffFieldId', diffFieldId.value)   // string 类型
  localStorage.setItem('appcode', appcode.value)   // string 类型

  

  // 加载bitable实例
  const { tableId, viewId } = await bitable.base.getSelection();
  const table = await bitable.base.getActiveTable();
  const view = await table.getViewById(viewId);

  // ## mode1: 全部记录
  // const RecordList = await view.getVisibleRecordIdList()

  // ## model2: 交互式选择记录 
  const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);

  // “节点最新时间”字段不同格式对应的处理
  const latestTimeField = await table.getFieldById(latestTimeFieldId.value)
  console.log("latestTimeField:", Object.getPrototypeOf(latestTimeField).type)  
  let latestTimeFieldType = Object.getPrototypeOf(latestTimeField).type


  
  

 
  for (let recordId of RecordList) {
     try {
       // TODO：在这里书写处理逻辑——数据请求、数据写入等
       let number = await getCellValueByRFIDS(recordId, numberFieldId.value)
       // 不同快递号格式的处理
       if (number[0].type == 'text') {
         number = number[0].text
       }
       if(number.length == 0){
         continue
       }
       console.log("快递号1：", number)

       const res = await getLocsInfo(number)
       console.log(res)
       // 请求错误处理
       if (res == 'ERROR') {
         await bitable.ui.showToast({
           toastType: 'error',
           message: '请求错误'
         })
         return
       }

       // 1. 物流节点最新时间
       console.log(res.list)
       if (latestTimeFieldType == 5) { // 时间格式  
         const dateObj = new Date(res.list[0].time);
         const timestamp = dateObj.getTime();
         await latestTimeField.setValue(recordId, timestamp);

       } else if (latestTimeFieldType == 1)  // 文本格式
         await table.setCellValue(latestTimeFieldId.value, recordId, [{ type: 'text', text: res.list[0].time }])


       // 2. 物流节点最新状态
       await table.setCellValue(latestStatusFieldId.value, recordId, [{ type: 'text', text: res.list[0].status }])

       // 3. 物流刷新结果
       let lastStatus = await getCellValueByRFIDS(recordId, latestStatusFieldId.value) // 上一次节点状态
       console.log(111)
       lastStatus = lastStatus[0].text
       console.log("lastStatus", lastStatus)
       let lastDiffText =  await getCellValueByRFIDS(recordId, diffFieldId.value) // 上一次diff结果
       console.log("lastDiffText No.1", lastDiffText)
       if (!lastDiffText)
         lastDiffText = " "
       else 
         lastDiffText = lastDiffText[0].text
       console.log("lastDiffText No.2", lastDiffText)

       const expName = res.expName  // 已处理
       const nowTime = getTime()  // 已处理
       const isStatusChange = (lastStatus == res.list[0].status ? '×' : '√') // 已处理
       const lastRefreshTime = getLastDiffTime(lastDiffText)  // 已处理

       const diffText = `${expName} 于 ${nowTime} 刷新。新变动 ${isStatusChange} 。上次刷新时间  ${lastRefreshTime}`

         // 3. 的最后一步，写回结果
       await table.setCellValue(diffFieldId.value, recordId, [{ type: 'text', text: diffText }])
       } catch (e){
         console.log("error:\n", e)
         await bitable.ui.showToast({
           toastType: 'warning',
           message: '跳过了一个错误的快递单号'
         })
       }
  }
  await bitable.ui.showToast({
    toastType: 'success',
    message: '处理完毕'
  })

}



// -- 辅助算法区域
// --001== 发送请求，获取物流信息
const getLocsInfo = async (number) => {
  const url = `${interfaceAddr.value}?no=${number}`;

  const config = {
    headers: {
      'Authorization': `APPCODE ${appcode.value}`,
      'Content-Type': 'application/json; charset=UTF-8'
    }
  };

  let res = ''
  await axios.get(url, config)
    .then((response) => {
      console.log(response)
      res = response
    }).catch(err => {
      console.log(err)
      // isDataWriten.value = 2
      // requestErrorInfo.value = err
    })

  console.log("请求结果", res.data.result)
  if (res.status === 200)
    return res.data.result
  else
    return "ERROR"
};
// --002== 依据 recordId 和 fieldId 获取 cell Value
const getCellValueByRFIDS = async (recordId, fieldId) => {
  const selection = await bitable.base.getSelection();
  const table = await bitable.base.getTableById(selection.tableId);
  const cellValue = await table.getCellValue(fieldId, recordId)

  return cellValue
}

// --003== 获取 2023/11/23 09:31 格式的当前时间
const getTime = () => {
  var currentDate = new Date();
  var year = currentDate.getFullYear();
  var month = currentDate.getMonth() + 1;
  var day = currentDate.getDate();
  var hours = currentDate.getHours();
  var minutes = currentDate.getMinutes();
  
  month = month < 10 ? '0' + month : month;
  day = day < 10 ? '0' + day : day;
  hours = hours < 10 ? '0' + hours : hours;
  minutes = minutes < 10 ? '0' + minutes : minutes;
  
  var formattedDate = year + '/' + month + '/' + day + ' ' + hours + ':' + minutes;
  return formattedDate
}
// --004== 防抖函数，辅助执行 “刷新物流信息” 按钮，闭包思想实现的
const debounce = (func) => {
  let timer;
  
  return function() {
    clearTimeout(timer);
    timer = setTimeout(func, 200);
  };
}
const debouncedFunction = debounce(writeData);

// --005== 正则提取，获取上一次的diff时间
const getLastDiffTime = (text) => {
  
  const regex = /\d{4}\/\d{2}\/\d{2} \d{2}:\d{2}/;
  const match = regex.exec(text);
  console.log("match", match)
  if (match && match.length > 0) {
    const targetTime = match[0];
    console.log(targetTime); // 输出：2023/11/23 09:31
    return targetTime
  } else {
    return '第一次刷新'
  }
  
}


onMounted(async () => {
  // 获取字段列表 -- start
  const selection = await bitable.base.getSelection()
  const table = await bitable.base.getTableById(selection.tableId)
  const view = await table.getViewById(selection.viewId)
  fieldListSeView.value = await view.getFieldMetaList()

  
  // 获取字段列表 -- end
  
  // 从缓存中获取数据 -- start
  if (localStorage.getItem('numberFieldId') !== null) {
    numberFieldId.value = localStorage.getItem('numberFieldId')
  }
  if (localStorage.getItem('latestTimeFieldId') !== null) {
    latestTimeFieldId.value = localStorage.getItem('latestTimeFieldId')
  }
  if (localStorage.getItem('latestStatusFieldId') !== null) {
    latestStatusFieldId.value = localStorage.getItem('latestStatusFieldId')
  }
  if (localStorage.getItem('diffFieldId') !== null) {
    diffFieldId.value = localStorage.getItem('diffFieldId')
  }
  if (localStorage.getItem('appcode') !== null) {
    appcode.value = localStorage.getItem('appcode')
  }
  // 从缓存中获取数据 -- end
});
    
</script>



<style scoped>
  
</style>

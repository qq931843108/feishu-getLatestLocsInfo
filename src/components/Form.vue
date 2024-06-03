<template>
  <el-alert style="margin-bottom: 20px;background-color: #e1eaff;color: #606266;"  :title="$t('alerts.selectNumberField')" type="info" />
  <el-link style="color: #3e75f5;" type="primary" :href="DOCX_LINK"
    target="_blank">👉 {{ $t('apiDescDocument') }}</el-link>

  <el-form ref="form" class="form" label-position="top" style="margin-top: 20px;">
    <el-form-item :label="$t('labels.api')" size="large" required>
      <span>{{ $t('labels.apiConfig.address') }}</span><el-input v-model="INTERFACE_ADDR" :placeholder="$t('placeholder.apiAddress')"></el-input>
      <span>{{ $t('labels.apiConfig.APPCODE') }}</span><el-input type="password" v-model="APPCODE" :placeholder="$t('placeholder.apiAppcode')"></el-input>
      
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

  <!-- DESCRIPTION  -->
  <div v-if="isProgressStarted" class="demo-progress" style="margin: 10px 0 0 0;">
    <div style="font-size: 14px; margin-bottom: 4px;">{{ $t('infoTip.process') }}</div>
    <el-progress :percentage="progressPercentage" />
  </div>

  <el-alert v-if="isProgressEnded" style="margin-top: 20px; background-color: #e1eaff;color: #606266;" :title="processResultDesc"  type="success" show-icon />
  


</template>

<script setup>
import { bitable, FieldType } from '@lark-base-open/js-sdk';
import { useI18n } from 'vue-i18n';
import { ref, onMounted } from 'vue';
import axios from 'axios';



// -- 数据区域

// 配置区域
const { t } = useI18n();
const DOCX_LINK = ref("https://jfsq6znqku.feishu.cn/wiki/Pr0kwAAn9iQMIakrZK1coFIynhf?from=from_copylink")
const INTERFACE_ADDR = ref('https://wuliu.market.alicloudapi.com/kdi')
const APPCODE = ref('')
const FieldTypeMap = {
    NotSupport: 0,
    Text: 1,
    Number: 2,
    SingleSelect: 3,
    MultiSelect: 4,
    DateTime: 5,
    Checkbox: 7,
    User: 11,
    Phone: 13,
    Url: 15,
    Attachment: 17,
    SingleLink: 18,
    Lookup: 19,
    Formula: 20,
    DuplexLink: 21,
    Location: 22,
    GroupChat: 23,
    Denied: 403,
    /**
     * 引用类型字段，前后端约定用10xx公共前缀开头
     */
    CreatedTime: 1001,
    ModifiedTime: 1002,
    CreatedUser: 1003,
    ModifiedUser: 1004,
    AutoNumber: 1005,
    Barcode: 99001,
    Progress: 99002,
    Currency: 99003,
    Rating: 99004,
    Email: 99005
}


// 核心数据
const numberFieldId = ref('')
const latestTimeFieldId = ref('')
const latestStatusFieldId = ref('')
const diffFieldId = ref('')

const diffInfo = ref('')


// 辅助数据
const fieldListSeView = ref([])
const isProgressStarted = ref(false)
const isProgressEnded = ref(false)
const progressPercentage = ref(1)
const processResultDesc = ref("")

let totalLocsNum = 0
let errorLocsNum = 0




// -- 核心算法区域
// --001== 表单最终按钮触发，数据写回
const writeData = async () => {

  // 检查必选信息是否已填写
  const checkRequiredRes = await checkIsRequiredInfoFilled()
  if (checkRequiredRes.isError)
    return

  // {User} 获取用户在表单中填写的各个字段的值
  const { numberFieldId, latestTimeFieldId, latestStatusFieldId, diffFieldId } = queryFormItemInfo()


   // 插件运行开始提示
  await showProcessTip("start")

  
  // 查询 Base SDK table、view 等实例，并获取当前表格的字段元信息
  const {table, view, existedFieldMetaList, tableId, viewId} = await queryBaseTableAndView()

  // 获取当前表格的记录列表
  const RecordList = await getRecordListByMode(2, tableId, viewId)

  
  // 校验字段格式
  const checkTypeRes = await checkAllOutputFieldType(numberFieldId, latestTimeFieldId, latestStatusFieldId, diffFieldId, table)
  if (checkTypeRes.isError)
    return


  // “节点最新时间”字段不同格式对应的处理
  const latestTimeField = await table.getFieldById(latestTimeFieldId)
  console.log("latestTimeField:", Object.getPrototypeOf(latestTimeField).type)  
  let latestTimeFieldType = Object.getPrototypeOf(latestTimeField).type


  for (let recordId of RecordList) {
    
    let numberCellValue = await getCellValueByRFIDS(recordId, numberFieldId)
    if(!numberCellValue) {
      continue
    }
    
    let number = numberCellValue[0].text
    console.log("number", number)

    if (number.match(/^SF/) && !number.match(/:\d{4}/)) {
      await handleErrorTip("", "SF")
      continue
    }
    

    totalLocsNum ++


    

    let res = await getLocsInfo(number)
    // 请求错误处理
    if (res.isError) {
      await handleErrorTip(res.errMsg, "request-error")
      return
    }
    res = res.data

    // 1. 物流节点最新时间
    console.log(res.list)
    if (latestTimeFieldType == 5) { // 时间格式  
      const dateObj = new Date(res.list[0].time);
      const timestamp = dateObj.getTime();
      await latestTimeField.setValue(recordId, timestamp);

    } else if (latestTimeFieldType == 1)  // 文本格式
      await table.setCellValue(latestTimeFieldId, recordId, [{ type: 'text', text: res.list[0].time }])


    // 2. 物流节点最新状态
    await table.setCellValue(latestStatusFieldId, recordId, [{ type: 'text', text: res.list[0].status }])

    // 3. 物流刷新结果
    let lastStatus = await getCellValueByRFIDS(recordId, latestStatusFieldId) // 上一次节点状态
    console.log(111)
    lastStatus = lastStatus[0].text
    console.log("lastStatus", lastStatus)
    let lastDiffText =  await getCellValueByRFIDS(recordId, diffFieldId) // 上一次diff结果
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
    await table.setCellValue(diffFieldId, recordId, [{ type: 'text', text: diffText }])
      
  }
  await showProcessTip("end")

}




/**
 * @query 查询表单填写的信息
 * @return {object} numberFieldId, latestTimeFieldId, latestStatusFieldId, diffFieldId
 */
const queryFormItemInfo = () => {
  setLocalStorage()

  return {INTERFACE_ADDR: INTERFACE_ADDR.value, APPCODE: APPCODE.value, numberFieldId: numberFieldId.value, latestTimeFieldId: latestTimeFieldId.value, latestStatusFieldId: latestStatusFieldId.value, diffFieldId: diffFieldId.value}
}


/**
 * @query(status) {插件运行状态通知}
 * @param {string} Tiptype 提示类型
 */
const showProcessTip = async (Tiptype) => {
  if (Tiptype === "start") {
    progressPercentage.value = 1
    isProgressStarted.value = true
    isProgressEnded.value = false

    const interval = setInterval(() => {
      // 随机增加进度，模拟加载过程
      progressPercentage.value += Math.floor(Math.random() * 10) 
      if (progressPercentage.value > 90) {
        clearInterval(interval);
      }
    }, 1000); // 每秒更新一次进度
    
    await bitable.ui.showToast({
      toastType: 'success',
      message: t('infoTip.start')
    })

  } else if (Tiptype === "end") {
    let endInfo = t('infoTip.end_sentence')
    const resDesc = endInfo.replace("totalLocsNum", totalLocsNum).replace("errorLocsNum", errorLocsNum)
    progressPercentage.value = 100
    processResultDesc.value = resDesc
    isProgressStarted.value = false
    isProgressEnded.value = true

    totalLocsNum = 0
    errorLocsNum = 0

    await bitable.ui.showToast({
      toastType: 'success',
      message: t('infoTip.end')
    })
  }
}

/**
 * @other 存入缓存
 */
const setLocalStorage = () => {
  localStorage.setItem('numberFieldId', numberFieldId.value)   // string 类型
  localStorage.setItem('latestTimeFieldId', latestTimeFieldId.value)   // string 类型
  localStorage.setItem('latestStatusFieldId', latestStatusFieldId.value)   // string 类型
  localStorage.setItem('diffFieldId', diffFieldId.value)   // string 类型
  localStorage.setItem('APPCODE', APPCODE.value)   // string 类型
}

/**
 * @query 依据不同的模式，查询表格记录： 1-全部获取  2-交互式选取
 * @param {number} mode 表格记录查询模式
 */
const getRecordListByMode = async (mode, tableId, viewId) => {
  if (mode == 1) {
    const RecordList = await view.getVisibleRecordIdList()
    return RecordList
  }
  else if (mode == 2) {
    const RecordList = await bitable.ui.selectRecordIdList(tableId, viewId);
    return RecordList
  }
}


/**
 * @query(check) {检查是否已填写了必要信息}
 * @return {object}
 */
const checkIsRequiredInfoFilled = async () => {
  // 检查是否填写了 Instagram 请求头信息
  if (!numberFieldId.value || !latestTimeFieldId.value || !latestStatusFieldId.value || !diffFieldId.value || !INTERFACE_ADDR.value || !APPCODE.value) {
    await handleErrorTip("", "empty_params")
    return {"isError": true}
  }

  return {"isError": false}
}



/**
 * @query(hanlde) {报错提示函数}
 * @param {string} errorMsg 错误提示
 * @param {string} errorType 错误类型
 */
const handleErrorTip = async (errorMsg, errorType) => {
  if (errorType === "request-error" || errorType === "field-type-error") {
    await bitable.ui.showToast({
      toastType: 'error',
      message: errorMsg
    })
  }

  else {
    await bitable.ui.showToast({
      toastType: 'error',
      message: t(`errorTip.${errorType}`)
    })
  }
  await showProcessTip("end")


}


/**
 * @query(hanlde) {检查输出字段类型}
 * @param {string} numberFieldId
 * @param {string} latestTimeFieldId
 * @param {string} latestStatusFieldId
 * @param {string} diffFieldId
 */
const checkAllOutputFieldType = async (numberFieldId, latestTimeFieldId, latestStatusFieldId, diffFieldId, table) => {
  const expectedFieldTypes = {
    [numberFieldId]: "Text",
    [latestTimeFieldId]: "DateTime",
    [latestStatusFieldId]: "Text",
    [diffFieldId]: "Text",
  };

  for (let fieldId in expectedFieldTypes) {
    const fieldType = expectedFieldTypes[fieldId];
    const fieldMeta = await getFieldMetaById(fieldId, table);
    const errorMsg = `⌜${fieldMeta.fieldName}⌟ ${t(`fieldTypeChecks.${fieldType}`)}`;

    if (fieldMeta.fieldType !== FieldTypeMap[fieldType]) {
      await handleErrorTip(errorMsg, "field-type-error");
      return { isError: true };
    }
  }

  return { isError: false };
};
  

/**
* @query {依据fieldId返回field的字段类型}
* @param {string} fieldId
* @return {string} fieldType
*/
const getFieldMetaById= async (fieldId, table) => {
  const fieldMeta = await table.getFieldMetaById(fieldId)

  const fieldName = fieldMeta.name
  const fieldType = fieldMeta.type
  
  return { fieldName, fieldType, fieldId }
}

/**
 * @query {sendAPI } 依据快递号，发送请求，获取快递信息
 * @param {string} number 快递号
 * @return {object} 返回接口请求的快递信息
 */
const getLocsInfo = async (number) => {
  const url = `${INTERFACE_ADDR.value}?no=${number}`;

  const config = {
    headers: {
      'Authorization': `APPCODE ${APPCODE.value}`,
      'Content-Type': 'application/json; charset=UTF-8'
    }
  };

  // 发送POST请求
  try {
      const res = await await axios.get(url, config);
      console.log("请求结果", res.data.result)
      return { data: res.data.result, isError: false };
  } catch (error) {
      console.error('Error:', error);
      if (error?.response?.data?.error)
        return { errorMsg: error.response.data.error, isError: true };
      else
        return { errorMsg: error.message, isError: true };
  }
};

/**
 * @query 依据 recordId 和 fieldId 获取 CellValue
 * @param {string} recordId 
 * @param {string} fieldId 
 * @return {string} cellValue
 */
const getCellValueByRFIDS = async (recordId, fieldId) => {
  const selection = await bitable.base.getSelection();
  const table = await bitable.base.getTableById(selection.tableId);
  const cellValue = await table.getCellValue(fieldId, recordId)

  return cellValue
}

/**
 * @query 获取 2023/11/23 09:31 格式的当前时间
 */
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


/**
 * @command {function} 防抖函数，辅助执行 “刷新物流信息” 按钮，闭包思想实现的
 * @param {function} func 防抖目标函数
 */
const debounce = (func) => {
  let timer;
  
  return function() {
    clearTimeout(timer);
    timer = setTimeout(func, 200);
  };
}
const debouncedFunction = debounce(writeData);

/**
 * 正则提取，获取上一次的diff时间
 * @param {string} text 上次更新的文本信息
 * @return {string} 如有，则提取时间信息，否则返回“第一次刷新”
 */
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

/**
 * @command {update} 从缓存中读取数据
 * @data {numberFieldId,latestTimeFieldId,latestStatusFieldId,diffFieldId,APPCODE}
 */
const setVariableFromLocalStorage = () => {
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
  if (localStorage.getItem('APPCODE') !== null) {
    APPCODE.value = localStorage.getItem('APPCODE')
  }
}

/**
 * @query {获取 SDK table、view、existedFieldMetaList 等信息}
 * @return {object} table, view, existedFieldMetaList
 */
const queryBaseTableAndView = async () => {
  const selection = await bitable.base.getSelection()
  const tableId = selection.tableId
  const viewId = selection.viewId
  const table = await bitable.base.getTableById(tableId)
  const view = await table.getViewById(viewId)
  const existedFieldMetaList = await table.getFieldMetaList();

  return {table, view, existedFieldMetaList, tableId, viewId}
}



onMounted(async () => {

  // 查询 Base SDK table、view 等实例，并获取当前表格的字段元信息
  const {table, view, existedFieldMetaList, tableId, viewId} = await queryBaseTableAndView()
  fieldListSeView.value = existedFieldMetaList
  
  // 从缓存中获取数据
  setVariableFromLocalStorage()
});
    
</script>



<style scoped>
  
</style>

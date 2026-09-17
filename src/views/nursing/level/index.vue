<template>
  <div class="app-container">
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="等级名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="请输入等级名称"
          clearable
          @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="请选择状态" clearable style="width: 200px;">
          <el-option
            v-for="dict in nursing_level_status"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="Plus"
          @click="handleAdd"
          v-hasPermi="['nursing:level:add']"
        >新增</el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <el-table v-loading="loading" :data="levelList">
      <el-table-column label="序号" type="index" width="60" align="center" />
      <el-table-column label="等级名称" align="center" prop="name" />
      <el-table-column label="护理计划" align="center" prop="planName">
        <template #default="scope">
          <span>{{ scope.row.planName || formatPlanName(scope.row.lplanId || scope.row.planId) }}</span>
        </template>
      </el-table-column>
      <el-table-column label="护理费用" align="center" prop="fee" />
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <el-tag :type="Number(scope.row.status) === 1 ? 'success' : 'danger'">
            {{ formatStatus(scope.row.status) }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="220" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['nursing:level:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['nursing:level:remove']">删除</el-button>
          <el-button link type="primary" :icon="Number(scope.row.status) === 0 ? 'Unlock' : 'Lock'" @click="handleStatus(scope.row)">
            {{ Number(scope.row.status) === 1 ? '禁用' : '启用' }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <pagination
      v-show="total>0"
      :total="total"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="100px">
        <el-row>
          <el-col :span="24">
            <el-form-item label="等级名称" prop="name">
              <el-input
                v-model="form.name"
                placeholder="请输入"
                maxlength="10"
                show-word-limit
              />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="护理计划" prop="lplanId">
              <el-select v-model="form.lplanId" placeholder="请设置" clearable style="width: 100%;">
                <el-option
                  v-for="item in planOptions"
                  :key="item.id || item.value"
                  :label="item.planName || item.name || item.label"
                  :value="String(item.id || item.value)"
                />
              </el-select>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="护理费用" prop="fee">
              <el-input-number
                v-model="form.fee"
                :precision="2"
                :min="0"
                :step="1"
                controls-position="right"
                placeholder="0.00"
              />
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="状态" prop="status">
              <el-radio-group v-model="form.status">
                <el-radio
                  v-for="dict in nursing_level_status"
                  :key="dict.value"
                  :label="Number(dict.value)"
                  :value="Number(dict.value)"
                >{{ dict.label }}</el-radio>
              </el-radio-group>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="等级说明" prop="description">
              <el-input
                v-model="form.description"
                type="textarea"
                placeholder="请输入"
                maxlength="50"
                show-word-limit
                :rows="3"
              />
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="cancel">取 消</el-button>
          <el-button type="primary" @click="submitForm">确 定</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Level">
import { listLevel, getLevel, delLevel, addLevel, updateLevel, getAllPlans } from "@/api/nursing/level"

const { proxy } = getCurrentInstance()

const levelList = ref([])
const planOptions = ref([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const total = ref(0)
const title = ref("")

const { nursing_level_status } = proxy.useDict('nursing_level_status')

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: undefined,
    status: undefined,
  },
  rules: {
    name: [
      { required: true, message: "等级名称不能为空", trigger: "blur" }
    ],
    lplanId: [
      { required: true, message: "护理计划不能为空", trigger: "change" }
    ],
    fee: [
      { required: true, message: "护理费用不能为空", trigger: "blur" }
    ],
    status: [
      { required: true, message: "状态不能为空", trigger: "change" }
    ],
    description: [
      { required: true, message: "等级说明不能为空", trigger: "blur" }
    ],
  }
})

const { queryParams, form, rules } = toRefs(data)

/** 查询所有护理计划列表 */
function getPlanList() {
  return getAllPlans().then(response => {
    planOptions.value = response.data || response.rows || []
  }).catch(error => {
    proxy.$modal.msgError("获取护理计划列表失败，请稍后重试")
  })
}

/** 格式化护理计划名称 */
function formatPlanName(planId) {
  if (!planId) return ''
  const item = planOptions.value.find(p => (p.id || p.value) == planId)
  return item ? (item.planName || item.name || item.label) : planId
}

/** 格式化状态文本 */
function formatStatus(status) {
  if (nursing_level_status.value && nursing_level_status.value.length) {
    const item = nursing_level_status.value.find(d => Number(d.value) === Number(status))
    if (item) return item.label
  }
  return Number(status) === 1 ? '启用' : '禁用'
}

/** 查询护理等级列表 */
function getList() {
  loading.value = true
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

/** 取消按钮 */
function cancel() {
  open.value = false
  reset()
}

/** 表单重置 */
function reset() {
  form.value = {
    id: null,
    name: null,
    lplanId: null,
    fee: 0.00,
    status: 1,
    description: null,
    remark: null
  }
  proxy.resetForm("levelRef")
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

/** 新增按钮操作 */
function handleAdd() {
  reset()
  getPlanList()
  open.value = true
  title.value = "新增护理等级"
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset()
  const _id = row.id
  getPlanList()
  getLevel(_id).then(response => {
    form.value = response.data
    form.value.status = Number(form.value.status)
    if (form.value.fee !== null && form.value.fee !== undefined) {
      form.value.fee = Number(form.value.fee)
    }
    // 对齐接口文档 lplanId 字段
    if (response.data.lplanId !== null && response.data.lplanId !== undefined) {
      form.value.lplanId = String(response.data.lplanId)
    } else if (response.data.planId !== null && response.data.planId !== undefined) {
      form.value.lplanId = String(response.data.planId)
    }
    open.value = true
    title.value = "修改护理等级"
  })
}

/** 状态切换操作 */
function handleStatus(row) {
  const isEnable = Number(row.status) === 1
  const text = isEnable ? '禁用' : '启用'
  const newStatus = isEnable ? 0 : 1
  proxy.$modal.confirm(`是否确认${text}护理等级名称为"${row.name}"的数据项？`).then(function() {
    return updateLevel({ id: row.id, status: newStatus })
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess(`${text}成功`)
  }).catch(() => {})
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (valid) {
      const submitData = {
        ...form.value,
        lplanId: form.value.lplanId || form.value.planId
      }
      if (form.value.id != null) {
        updateLevel(submitData).then(() => {
          proxy.$modal.msgSuccess("修改成功")
          open.value = false
          getList()
        })
      } else {
        addLevel(submitData).then(() => {
          proxy.$modal.msgSuccess("新增成功")
          open.value = false
          getList()
        })
      }
    }
  })
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _id = row.id
  proxy.$modal.confirm('是否确认删除护理等级名称为"' + row.name + '"的数据项？').then(function() {
    return delLevel(_id)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}

onMounted(() => {
  getPlanList()
})

onActivated(() => {
  getPlanList()
})

getList()
</script>

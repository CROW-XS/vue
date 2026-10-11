<!-- src/components/EmpList.vue -->
<template>
  <!-- 顶部菜单 -->
  <div class="top-bar">
    <div class="logo"><span>tlias</span> 智能学习辅助系统</div>
    <div class="logout" @click="showLogoutModal">
      <svg viewBox="0 0 24 24">
        <path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"/>
      </svg>
      退出登录
    </div>
  </div>

  <div class="layout">
    <!-- 左侧菜单 -->
    <div class="side-menu">
      <div class="menu-group" @click="toggleMenu">
        系统信息管理
        <span class="arrow" v-show="menuOpen">▼</span>
        <span class="arrow" v-show="!menuOpen">▶</span>
      </div>
      <ul class="sub-menu" v-show="menuOpen">
        <li class="active">员工管理</li>
      </ul>
    </div>

    <!-- 主内容 -->
    <div class="main">
      <!-- 查询条件卡片 -->
      <div class="search-card">
        <label for="nameInput">姓名</label>
        <input
            type="text"
            id="nameInput"
            placeholder="请输入员工姓名"
            v-model="searchForm.name"
            @keyup.enter="doQuery"
        />
        <label for="genderSelect">性别</label>
        <select id="genderSelect" v-model="searchForm.gender">
          <option value="">请选择</option>
          <option value="1">男</option>
          <option value="2">女</option>
        </select>
        <label for="jobSelect">职位</label>
        <select id="jobSelect" v-model="searchForm.job">
          <option value="">请选择</option>
          <option value="1">班主任</option>
          <option value="2">讲师</option>
          <option value="3">学工主管</option>
          <option value="4">教研主管</option>
          <option value="5">咨询师</option>
        </select>
        <button class="btn btn-query" @click="doQuery">查询</button>
        <button class="btn btn-reset" @click="doReset">清空</button>
      </div>

      <!-- 加载中 / 错误提示 -->
      <div v-if="loading" class="loading">⏳ 正在加载员工数据...</div>
      <div v-else-if="error" class="error-msg">❌ {{ error }}</div>

      <!-- 员工列表表格 -->
      <div class="table-card" v-else>
        <table id="empTable">
          <thead>
          <tr>
            <th style="width:125px;">姓名</th>
            <th style="width:90px;">性别</th>
            <th style="width:138px;">头像</th>
            <th style="width:154px;">职位</th>
            <th style="width:172px;">入职日期</th>
            <th style="width:225px;">最后操作时间</th>
            <th style="width:247px;">操作</th>
          </tr>
          </thead>
          <tbody id="empTbody">
          <tr v-for="emp in filteredEmpList" :key="emp.id">
            <td>{{ emp.name }}</td>
            <td>{{ emp.gender === 1 ? '男' : '女' }}</td>
            <td>
              <img
                  class="avatar-img"
                  :src="emp.image"
                  alt="头像"
                  @error="onImgError"
              />
            </td>
            <td>{{ jobMap[emp.job] || '未知' }}</td>
            <td>{{ emp.entrydate }}</td>
            <td>{{ formatDateTime(emp.updatetime) }}</td>
            <td class="op-links">
              <a @click="editEmployee(emp)">编辑</a>
              <a @click="askDelete(emp)">删除</a>
            </td>
          </tr>
          <tr v-if="filteredEmpList.length === 0" class="no-data">
            <td colspan="7">暂无符合条件的员工数据</td>
          </tr>
          </tbody>
        </table>
      </div>

      <!-- 页脚 -->
      <div class="footer">
        <p>江苏传智播客教育科技股份有限公司</p>
        <p>版权所有Copyright 2006-2024&nbsp; All Rights Reserved</p>
      </div>
    </div>
  </div>

  <!-- 退出确认对话框 -->
  <div class="modal-mask" :class="{ show: logoutModalVisible }">
    <div class="modal">
      <div class="modal-title">退出登录</div>
      <div class="modal-body">您确定要退出登录吗？</div>
      <div class="modal-footer">
        <button class="btn btn-confirm" @click="logoutModalVisible = false">确定</button>
        <button class="btn btn-cancel" @click="logoutModalVisible = false">取消</button>
      </div>
    </div>
  </div>

  <!-- 删除员工对话框 -->
  <div class="modal-mask" :class="{ show: deleteModalVisible }">
    <div class="modal">
      <div class="modal-title">删除员工</div>
      <div class="modal-body">您确定要删除该员工吗 ?</div>
      <div class="modal-footer">
        <button class="btn btn-confirm" @click="confirmDelete">确定</button>
        <button class="btn btn-cancel" @click="closeDeleteModal">取消</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

// ---------- 数据定义 ----------
const empList = ref([])
const loading = ref(false)
const error = ref('')

const searchForm = reactive({
  name: '',
  gender: '',
  job: ''
})

const jobMap = {
  '1': '班主任',
  '2': '讲师',
  '3': '学工主管',
  '4': '教研主管',
  '5': '咨询师'
}

const menuOpen = ref(true)
const logoutModalVisible = ref(false)
const deleteModalVisible = ref(false)
const pendingDeleteId = ref(null)

// ---------- 核心方法：从服务器获取数据 ----------
const fetchEmployees = async () => {
  loading.value = true
  error.value = ''

  try {
    const params = new URLSearchParams()
    if (searchForm.name.trim() !== '') {
      params.append('name', searchForm.name.trim())
    }
    if (searchForm.gender !== '') {
      params.append('gender', searchForm.gender)
    }
    if (searchForm.job !== '') {
      params.append('job', searchForm.job)
    }

    const baseUrl = 'https://web-server.itheima.net/emps/list'
    const queryString = params.toString()
    const url = queryString ? `${baseUrl}?${queryString}` : baseUrl

    console.log('请求URL:', url)

    const response = await fetch(url)
    if (!response.ok) {
      throw new Error(`服务器响应异常 (状态码: ${response.status})`)
    }

    const result = await response.json()
    console.log('服务器返回数据:', result)

    // 兼容多种返回结构
    let dataArray = []
    if (Array.isArray(result)) {
      dataArray = result
    } else if (result.data && Array.isArray(result.data)) {
      dataArray = result.data
    } else if (result.code === 1 && result.data) {
      dataArray = result.data
    } else {
      console.warn('返回数据格式未识别:', result)
      dataArray = []
    }

    // 规范化字段
    empList.value = dataArray.map((item) => ({
      id: item.id,
      name: item.name,
      image: item.image || '',
      gender: typeof item.gender === 'string' ? parseInt(item.gender, 10) : item.gender,
      job: item.job ? String(item.job) : '',
      entrydate: item.entrydate,
      updatetime: item.updatetime
    }))
  } catch (err) {
    console.error('获取员工数据失败:', err)
    error.value = `加载员工数据失败: ${err.message}。请检查网络或稍后重试。`
    empList.value = []
  } finally {
    loading.value = false
  }
}

// ---------- 生命周期钩子 ----------
onMounted(() => {
  fetchEmployees()
})

// ---------- 计算属性：按最后操作时间倒序 ----------
const filteredEmpList = computed(() => {
  let list = empList.value
  return [...list].sort((a, b) => {
    const timeA = a.updatetime ? new Date(a.updatetime) : 0
    const timeB = b.updatetime ? new Date(b.updatetime) : 0
    return timeB - timeA
  })
})

// ---------- 方法定义 ----------
const formatDateTime = (datetimeStr) => {
  if (!datetimeStr) return ''
  return datetimeStr.replace('T', ' ')
}

// 图片加载失败时的兜底（对应原 onerror 内联写法）
const onImgError = (e) => {
  e.target.src =
      "data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='58' height='33' viewBox='0 0 58 33'%3E%3Crect x='1' y='1' width='56' height='31' rx='2' fill='%23ffffff' stroke='%23cccccc'/%3E%3Ccircle cx='32.6' cy='10.4' r='2.5' fill='%23cccccc'/%3E%3Cpath d='M3 30 L3 15.2 L16 8.4 L34.2 20.8 L44.6 12.4 L55 20.8 L55 30 L3 30 Z' fill='%23169bd5'/%3E%3C/svg%3E"
}

const doQuery = () => {
  fetchEmployees()
}

const doReset = () => {
  searchForm.name = ''
  searchForm.gender = ''
  searchForm.job = ''
  fetchEmployees()
}

const editEmployee = (emp) => {
  alert(`编辑员工：${emp.name} (ID: ${emp.id})`)
}

const askDelete = (emp) => {
  pendingDeleteId.value = emp.id
  deleteModalVisible.value = true
}

const closeDeleteModal = () => {
  deleteModalVisible.value = false
  pendingDeleteId.value = null
}

const confirmDelete = () => {
  if (pendingDeleteId.value !== null) {
    empList.value = empList.value.filter((emp) => emp.id !== pendingDeleteId.value)
  }
  closeDeleteModal()
}

const showLogoutModal = () => {
  logoutModalVisible.value = true
}

const toggleMenu = () => {
  menuOpen.value = !menuOpen.value
}
</script>

<style scoped>
/* ===== 原 HTML 中 <style> 的全部内容，粘贴到这里 ===== */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  font-family: "Microsoft YaHei", "SimSun", Arial, sans-serif;
  background-color: #f0f2f5;
  color: #333;
}
.top-bar {
  background-color: #81d3f8;
  height: 46px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
}
.top-bar .logo {
  color: #ffffff;
  font-size: 26px;
  font-weight: bold;
  letter-spacing: 1px;
}
.top-bar .logo span {
  font-family: Consolas, "Courier New", monospace;
  font-size: 28px;
}
.top-bar .logout {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #ffffff;
  font-size: 14px;
  cursor: pointer;
  border: 1px solid rgba(121, 121, 121, 0.6);
  padding: 5px 18px;
  user-select: none;
}
.top-bar .logout:hover {
  background-color: rgba(255, 255, 255, 0.25);
}
.top-bar .logout svg {
  width: 18px;
  height: 18px;
  fill: #ffffff;
}
.layout {
  display: flex;
  min-height: calc(100vh - 46px);
}
.side-menu {
  width: 210px;
  background-color: #2d3a4b;
  flex-shrink: 0;
  padding-top: 10px;
}
.side-menu .menu-group {
  color: #ffffff;
  font-size: 16px;
  padding: 12px 20px;
  cursor: pointer;
}
.side-menu .menu-group .arrow {
  float: right;
  font-size: 12px;
  transition: transform 0.2s;
}
.side-menu .sub-menu {
  list-style: none;
}
.side-menu .sub-menu li {
  padding: 11px 20px 11px 38px;
  font-size: 14px;
  color: #c8cfd8;
  cursor: pointer;
}
.side-menu .sub-menu li:hover {
  color: #ffffff;
  background-color: #3a4a5f;
}
.side-menu .sub-menu li.active {
  color: #ffffff;
  background-color: #02a7f0;
}
.main {
  flex: 1;
  padding: 20px;
}
.search-card {
  background-color: #ffffff;
  padding: 16px 20px;
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  border-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}
.search-card label {
  font-size: 14px;
  color: #333;
  padding-left: 8px;
}
.search-card input[type="text"],
.search-card select {
  height: 38px;
  border: 1px solid #797979;
  border-radius: 3px;
  font-size: 14px;
  color: #7f7f7f;
  padding: 0 10px;
  background-color: #ffffff;
  outline: none;
}
.search-card input[type="text"] {
  width: 220px;
}
.search-card input[type="text"]:focus,
.search-card select:focus {
  border-color: #02a7f0;
}
.search-card select {
  width: 150px;
}
.btn {
  height: 35px;
  min-width: 70px;
  border: none;
  border-radius: 3px;
  color: #ffffff;
  font-size: 14px;
  cursor: pointer;
  padding: 0 14px;
}
.btn-query {
  background-color: #02a7f0;
  margin-left: 10px;
}
.btn-query:hover {
  background-color: #0394d6;
}
.btn-reset {
  background-color: #aaaaaa;
}
.btn-reset:hover {
  background-color: #999999;
}
.table-card {
  background-color: #ffffff;
  border-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
  padding: 0;
  overflow-x: auto;
}
table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
  min-width: 1151px;
}
table th,
table td {
  border: 1px solid #cccccc;
  height: 40px;
  text-align: center;
  padding: 4px 8px;
}
table thead th {
  background-color: #f2f2f2;
  color: #333;
  font-weight: bold;
  height: 39px;
}
table tbody tr {
  background-color: #fff;
}
.avatar-img {
  width: 58px;
  height: 33px;
  object-fit: cover;
  border-radius: 2px;
  border: 1px solid #ccc;
  display: inline-block;
  vertical-align: middle;
}
.op-links {
  color: #ff9900;
}
.op-links a {
  color: #ff9900;
  text-decoration: none;
  cursor: pointer;
  margin: 0 10px;
}
.op-links a:hover {
  text-decoration: underline;
}
.no-data td {
  color: #999;
  height: 120px;
}
.footer {
  background-color: #d7d7d7;
  text-align: center;
  padding: 16px 0;
  margin-top: 20px;
  color: #34495e;
  font-size: 13px;
  line-height: 24px;
  border-radius: 4px;
}
.modal-mask {
  display: none;
  position: fixed;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.4);
  z-index: 100;
  align-items: center;
  justify-content: center;
}
.modal-mask.show {
  display: flex;
}
.modal {
  background-color: #ffffff;
  width: 400px;
  border-radius: 6px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}
.modal-title {
  background-color: #02a7f0;
  color: #ffffff;
  font-size: 16px;
  padding: 12px 18px;
}
.modal-body {
  padding: 26px 18px;
  font-size: 15px;
  color: #333;
  text-align: center;
}
.modal-footer {
  padding: 0 18px 18px;
  text-align: center;
}
.modal-footer .btn {
  margin: 0 10px;
}
.btn-confirm {
  background-color: #02a7f0;
}
.btn-confirm:hover {
  background-color: #0394d6;
}
.btn-cancel {
  background-color: #aaaaaa;
}
.btn-cancel:hover {
  background-color: #999999;
}
#empTbody tr:not(.no-data):hover {
  background-color: #f2e2e2 !important;
}
.loading,
.error-msg {
  text-align: center;
  padding: 40px 20px;
  font-size: 16px;
  color: #666;
  background: #fff;
  border-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
}
.error-msg {
  color: #d9534f;
  background-color: #fdf7f7;
  border-left: 4px solid #d9534f;
}
</style>
<template>
  <div class="page-container">
    <div class="inner-container">
      <h1 id="selectionTitle"></h1>
      <div id="selectionTip" style="color: red;"></div>
      <div class="search-container">
        <input type="text" id="courseNameInput" placeholder="请输入课程名称" class="search-input">
        <input type="text" id="collegeInput" placeholder="请输入开课学院" class="search-input">
        <input type="text" id="teacherInput" placeholder="请输入开课教师" class="search-input">
        <select id="dayInput" class="search-input" v-model="daySearch" style="margin-right:10px;">
          <option value="">所有日期</option>
          <option value="一">周一</option>
          <option value="二">周二</option>
          <option value="三">周三</option>
          <option value="四">周四</option>
          <option value="五">周五</option>
          <option value="六">周六</option>
          <option value="日">周日</option>
        </select>
        <select id="timeInput" class="search-input" v-model="timeSearch" style="margin-right:10px;">
          <option value="">所有时间</option>
          <option value="1">8:00-8：50</option>
          <option value="2">9:00-9：50</option>
          <option value="3">10:00-10:50</option>
          <option value="4">11:00-11:50</option>
          <option value="5">13:00-13:50</option>
          <option value="6">14:00-14:50</option>
          <option value="7">15:00-15:50</option>
          <option value="8">16:00-16:50</option>
          <option value="9">17:00-17:50</option>
          <option value="10">19:00-19:50</option>
          <option value="11">20:00-20:50</option>
          <option value="12">21:00-21:50</option>
        </select>
        <button @click="searchCourses" class="search-btn">查询</button>
        <input type="checkbox" id="onlyAvailable" v-model="onlyAvailable" @change="searchCourses" style="margin-left:15px;">
        <label for="onlyAvailable" style="margin-left:5px;">只查看有余量教学班</label>
      </div>
      <div id="searchResult"></div>
      <table id="courseTable">
        <thead>
        <tr>
          <th>课程名称</th>
          <th>开课学期</th>
          <th>课程时间</th>
          <th>开课学院</th>
          <th>开课教师</th>
          <th>课程容量</th>
          <th>选课人数</th>
          <th>操作</th>
        </tr>
        </thead>
        <tbody id="courseTableBody">
        <tr v-for="course in courses" :key="course.name + course.time">
          <td>{{ course.name }}</td>
          <td>{{ course.semester }}</td>
          <td>{{ course.time }}</td>
          <td>{{ course.college }}</td>
          <td>{{ course.teacher }}</td>
          <td>{{ course.capacity }}</td>
          <td>{{ course.candidate }}</td>
          <td>
            <button v-if="course.is_selected" @click="dropCourse(course.name,course.id)">退课</button>
            <button v-else @click="selectCourse(course.name,course.id)" :disabled="!course.available">选择</button>
          </td>
        </tr>
        </tbody>
      </table>
      <div v-if="showModal" class="modal-overlay">
        <div class="modal-content">
          <h2>提示</h2>
          <p>当前不在选课时间范围内，请在选课时间内进行操作</p>
          <button @click="showModal = false">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import {getCurrentUserId, getCurrentUserType} from '/src/infoModule/src/function/CurrentUser.ts'
export default {
  data() {
    return {
      PRIMARY_SELECTION_START: null,
      PRIMARY_SELECTION_END: null,
      SUPPLEMENTARY_SELECTION_START: null,
      SUPPLEMENTARY_SELECTION_END: null,
      selectedCourses: [],
      courses: [],
      onlyAvailable: false ,// 新增：只查看有余量教学班的选项
      daySearch:"",
      timeSearch:"",
      selectAvailable:false,
      userId: null,
      showModal: false, // 控制模态框显示
    };
  },
  async mounted() {
    try {
      const userId = await getCurrentUserId();
      if (!userId) {
        alert('用户未登录，请重新登录');
        // 可以跳转到登录页或其他处理
        return;
      }
      this.userId = userId;
      // 然后执行其他需要 userId 的操作
      await this.getselectionTime();
      await this.searchCourses();
    } catch (error) {
      console.error('获取用户ID失败:', error);
      alert('无法获取用户信息，请刷新页面重试');
    }
  },
  methods: {
    async searchCourses() {
      const courseName = document.getElementById('courseNameInput').value;
      const college = document.getElementById('collegeInput').value;
      const teacher = document.getElementById('teacherInput').value;

      try {
        // 把筛选条件传给queryCourseDatabase
        this.courses = await this.queryCourseDatabase(courseName, college, teacher);
      } catch (error) {
        alert('查询失败：' + error.message);
      }
    },
    async getselectionTime(){
      // 1. 先获取选课时间
      try {
        const response = await fetch(`http://localhost:8083/student/${this.userId}/getSelectionTime`);
        if (!response.ok) throw new Error('获取选课时间失败');
        const times = await response.json();
        // 将字符串转成Date对象
        this.PRIMARY_SELECTION_START = new Date(times.PRIMARY_SELECTION_START);
        this.PRIMARY_SELECTION_END = new Date(times.PRIMARY_SELECTION_END);
        this.SUPPLEMENTARY_SELECTION_START = new Date(times.SUPPLEMENTARY_SELECTION_START);
        this.SUPPLEMENTARY_SELECTION_END = new Date(times.SUPPLEMENTARY_SELECTION_END);
        const now = new Date();
        const titleElement = document.getElementById('selectionTitle');
        const tipElement = document.getElementById('selectionTip');
        if (now >= this.PRIMARY_SELECTION_START && now <= this.PRIMARY_SELECTION_END) {
          titleElement.textContent = '初选课程';
          tipElement.textContent = '当前处于初选时间内';
          this.selectAvailable = true; // 初选时间可选
        } else if (now >= this.SUPPLEMENTARY_SELECTION_START && now <= this.SUPPLEMENTARY_SELECTION_END) {
          titleElement.textContent = '补选课程';
          tipElement.textContent = '当前处于补选时间内';
          this.selectAvailable = true; // 补选时间可选
        } else {
          titleElement.textContent = '非选课时间';
          tipElement.textContent = '当前不在选课时间范围内';
          this.selectAvailable = false; // 非选课时间不可选
        }
      } catch (err) {
        alert('读取选课时间失败：' + err.message);
      }
    },
    // 修正：接收筛选参数
    async queryCourseDatabase(courseName, college, teacher) {
      try {
        const params = new URLSearchParams();
        const response = await fetch(`http://localhost:8083/student/${this.userId}/getSections`);
        if (!response.ok) throw new Error('网络错误');
        const data = await response.json();
        // 先映射后筛选
        const mappedData = data.map(item => ({
          id: item.sec_id,
          name: item.title,
          semester:item.semester,
          time: item.time,
          timeIds: Array.isArray(item.timeIds) ? item.timeIds : JSON.parse(item.timeIds),
          college: item.dept_name,
          teacher: item.teacher,
          capacity: item.capacity,
          candidate: item.capacity - item.left_capacity,
          available: item.left_capacity > 0,
          is_selected: item.is_selected
        }));
        return mappedData.filter(course => {
          const matchCourseName = !courseName || course.name.includes(courseName);
          const matchCollege = !college || course.college.includes(college);
          const matchTeacher = !teacher || course.teacher.includes(teacher);
          const matchAvailable = !this.onlyAvailable || course.capacity> course.candidate; // 只在选中“只查看有余量教学班”时才检查available
          const matchDay = !this.daySearch || course.time.includes(this.daySearch);
          // timeSearch 为空或被包含即可
          const matchTime =
              !this.timeSearch ||
              course.timeIds.includes(Number(this.timeSearch))
              || course.timeIds.includes(Number(this.timeSearch) + 12)
              || course.timeIds.includes(Number(this.timeSearch) + 24)
              || course.timeIds.includes(Number(this.timeSearch) + 36)
              || course.timeIds.includes(Number(this.timeSearch) + 48)
              || course.timeIds.includes(Number(this.timeSearch) + 60)
              || course.timeIds.includes(Number(this.timeSearch) + 72);
          return matchCourseName && matchCollege && matchTeacher && matchAvailable && matchDay && matchTime;
        });
      } catch (err) {
        throw err;
      }
    },
    async selectCourse(courseName, courseId) {
      if(!this.selectAvailable) {
        this.showModal=true;
        return;
      }
      const confirmSelection = confirm(`是否确认选择 ${courseName} 课程？`);
      if (confirmSelection) {
        try {
          const params = new URLSearchParams();
          const response = await fetch(`http://localhost:8083/student/${this.userId}/chooseCourse/${courseId}`);
          if (!response.ok){
            // 先拿到后端返回的字符串
            const errorMsg = await response.text();
            throw new Error(errorMsg);
          }
          this.getselectionTime();
          this.searchCourses();
          alert('选课成功');
        } catch (error) {
          alert('选课失败：' + error.message);
        }
      }
    },
    async dropCourse(courseName, courseId) {
      if(!this.selectAvailable) {
        this.showModal=true;
        return;
      }
      const confirmDrop = confirm(`是否确认退选 ${courseName} 课程？`);
      if (confirmDrop) {
        try {
          const params = new URLSearchParams();
          const response = await fetch(`http://localhost:8083/student/${this.userId}/dropCourse/${courseId}`);
          if (!response.ok){
            // 先拿到后端返回的字符串
            const errorMsg = await response.text();
            throw new Error(errorMsg);
          }
          this.getselectionTime();
          this.searchCourses();
          alert('退课成功');
        } catch (error) {
          alert('退课失败：' + error.message);
        }
      }
    }
  }
};
</script>

<style scoped>
.page-container {
  width: 100%;
  flex-grow: 1;
  padding: 20px;
  box-sizing: border-box;
}

.inner-container {
  background-color: #f8f9fa;
  padding: 30px 40px 40px;
  border-radius: 8px;
}

.inner-container h1 {
  font-size: 30px;
  font-weight: 600;
  color: #0d47a1;
  margin: 0;
}

/* 表格样式 */
table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 20px;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  background-color: white;
}

th,
td {
  padding: 10px;
  border: 1px solid #ddd;
  text-align: center;
}

th {
  background-color: #0d47a1;
  color: white;
  font-weight: bold;
}

/* 搜索框和按钮样式 */
.search-container {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.search-input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  margin-right: 10px;
}

.search-btn {
  padding: 10px 20px;
  background-color: #0d47a1;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.search-btn:hover {
  background-color: #1565c0;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.modal-content {
  background: #fff;
  padding: 32px 40px;
  border-radius: 8px;
  box-shadow: 0 4px 20px #0002;
  min-width: 280px;
  text-align: center;
  border: 1px solid #ccc; /* 这里加边框 */
}
</style>
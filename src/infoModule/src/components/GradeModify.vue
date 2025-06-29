<template>
  <div v-if="showTeacherSection" >
    <!-- 教师成绩管理部分 -->
    <div class="header">
      <h1>成绩管理</h1>
      <el-input v-model="searchKeyword" placeholder="输入学生或课程信息" style="width: 300px" />
    </div>

    <el-card>
      <el-table :data="filteredGrades" border>
        <el-table-column prop="id" label="id" />
        <el-table-column prop="studentId" label="学号" width="120" />
        <el-table-column prop="studentName" label="姓名" width="120" />
        <el-table-column prop="courseId" label="课程ID" width="120" />
        <el-table-column prop="courseName" label="课程名称" />
        <el-table-column prop="name" label="类型" />
        <el-table-column prop="grade" label="成绩" width="100" />
        <el-table-column prop="status" label="状态" width="120">
          <template #default="{ row }">
            <el-tag :type="statusColor[row.status]">{{ row.status }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="180" fixed="right">
          <template #default="{ row }">
            <el-button
                type="primary"
                size="small"
                @click="openApplyDialog(row)"
                :disabled="row.status !== '已确认'"
            >
              申请修改
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
  </div>

  <!-- 申请修改对话框 -->
  <el-dialog v-model="applyDialogVisible" title="成绩修改申请" width="500px">
    <el-form :model="applyForm" label-width="80px">
      <el-form-item label="学生姓名">
        <el-input v-model="applyForm.studentName" disabled />
      </el-form-item>
      <el-form-item label="课程名称">
        <el-input v-model="applyForm.courseName" disabled />
      </el-form-item>
      <el-form-item label="原成绩">
        <el-input v-model="applyForm.originalGrade" disabled />
      </el-form-item>
      <el-form-item label="新成绩" prop="newGrade" required>
        <el-input-number v-model="applyForm.newGrade" :min="0" :max="100" />
      </el-form-item>
      <el-form-item label="修改理由" prop="reason" required>
        <el-input v-model="applyForm.reason" type="textarea" />
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="applyDialogVisible = false">取消</el-button>
      <el-button type="primary" @click="submitApplication">提交申请</el-button>
    </template>
  </el-dialog>

  <!-- 管理员审核部分 -->
  <div v-if="showAdminSection">
    <div class="header">
      <h1>待审核申请</h1>
    </div>
    <el-card>
      <el-table :data="pendingApplications" border>
        <el-table-column prop="studentId" label="学号" width="120" />
        <el-table-column prop="studentName" label="姓名" width="120" />
        <el-table-column prop="courseName" label="课程" />
        <el-table-column prop="name" label="类型"/>
        <el-table-column label="成绩变化" width="120">
          <template #default="{ row }">
            {{ row.originalGrade }} → {{ row.newGrade }}
          </template>
        </el-table-column>
        <el-table-column prop="reason" label="申请理由" />
        <el-table-column prop="applyTime" label="申请时间" width="180" />
        <el-table-column label="操作" width="180" fixed="right">
          <template #default="{ row }">
            <el-button type="success" size="small" @click="handleAudit(row, true)">
              通过
            </el-button>
            <el-button type="danger" size="small" @click="handleAudit(row, false)">
              拒绝
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
  </div>
</template>

<script setup lang="ts">
import "../assets/pages_styles.css";
import { ref, computed, onMounted } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import axios from '../plugins/axios.ts'
import {getCurrentUserId, getCurrentUserType} from "../function/CurrentUser.ts";

// 定义GradeModifyDTO相关接口
interface GradeStatusDTO {
  id: string;
  studentId: number;
  studentName: string;
  courseId: number;
  courseName: string;
  name: string;
  grade: number;
  status: '已确认' | '待审核' | '已修改' | '已拒绝';
}

interface ApplicationDTO {
  id: string;
  gradeId: number;
  studentId: number;
  studentName: string;
  courseName: string;
  name : string;
  originalGrade: number;
  newGrade: number;
  reason: string;
  status: 'pending' | 'approved' | 'rejected';
  applyTime: string;
  checkTime?: string;
}

// 后端DTO定义
interface GradeChangeDTO {
  gradeChangeId: number;
  takesId: number;
  teacherId: number | null;
  result: boolean | null;
  name: string;
  newGrade: number;
  applyTime: string;
  checkTime: string | null;
  gradeId: number | null;
  studentId?: number;
  studentName?: string;
  courseId?: number;
  courseName?: string;
  originalGrade?: number;
  reason?: string;
}

const statusColor: Record<string, string>= {
  '已确认': 'success',
  '待审核': 'warning',
  '已修改': '',
  '已拒绝': 'danger'
}

// 状态定义
const gradeList = ref<GradeStatusDTO[]>([]);
const applications = ref<ApplicationDTO[]>([]);
const searchKeyword = ref('');
const applyDialogVisible = ref(false);
const showAdminSection = ref(false); // 根据实际用户角色切换
const showTeacherSection = ref(false); // 根据实际用户角色切换
const teacherId = ref<number|null>(null);



const applyForm = ref({
  gradeId: 0,
  studentName: '',
  courseName: '',
  name: '',
  originalGrade: 0,
  newGrade: 0,
  reason: ''
});

// 计算属性
const filteredGrades = computed(() => {
  return gradeList.value.filter(g =>
      g.studentName.includes(searchKeyword.value) ||
      g.courseName.includes(searchKeyword.value) ||
      g.courseId.toString().includes(searchKeyword.value))
});

const pendingApplications = computed(() => {
  return applications.value.filter(app => app.status === 'pending');
});

// 页面加载时获取数据
onMounted(async () => {
  try {
    const userType = await getCurrentUserType();
    if (userType === "ROLE_ADMIN"){
      showAdminSection.value = true;
      await fetchApplications()
    }
    else if (userType === "ROLE_TEACHER"){
      showTeacherSection.value = true;
      teacherId.value = await getCurrentUserId();
      await fetchGradeList();
    }
  } catch (error) {
    console.error('获取数据失败', error);
  }
});

// 获取成绩列表
const fetchGradeList = async () => {
  try {
    const response = await axios.get(`/api/grades/teacher/${teacherId.value}`);
    //console.log('后端原始响应 (response.data):', JSON.stringify(response.data, null, 2));
    // 转换数据格式
    gradeList.value = response.data.map((item: GradeStatusDTO) => ({
      id: item.id,
      studentId: item.studentId,
      studentName: item.studentName,
      courseId: item.courseId,
      courseName: item.courseName,
      name: item.name,
      grade: item.grade,
      status: item.status
    }));
  } catch (error) {
    console.error('获取成绩列表失败', error);
  }
};

// 获取申请列表
const fetchApplications = async () => {
  try {
    const response = await axios.get(`/api/grade-changes/pending`);

    // 转换数据格式
    applications.value = response.data.map((item: GradeChangeDTO) => ({
      id: item.gradeChangeId.toString(),
      gradeId: item.gradeId || 0,
      studentId: item.studentId || 0,
      studentName: item.studentName || '',
      courseName: item.courseName || '',
      name: item.name || '',
      originalGrade: item.originalGrade || 0,
      newGrade: item.newGrade,
      reason: item.reason || '',
      status: getApplicationStatus(item),
      applyTime: item.applyTime,
      checkTime: item.checkTime
    }));
  } catch (error) {
    console.error('获取申请列表失败', error);
  }
};


// 转换申请状态
const getApplicationStatus = (application: GradeChangeDTO): 'pending' | 'approved' | 'rejected' => {
  if (application.result === null) {
    return 'pending';
  } else if (application.result === true) {
    return 'approved';
  } else {
    return 'rejected';
  }
};

// 打开申请对话框
const openApplyDialog = (grade: GradeStatusDTO) => {
  applyForm.value = {
    gradeId: parseInt(grade.id),
    studentName: grade.studentName,
    courseName: grade.courseName,
    name: grade.name,
    originalGrade: grade.grade,
    newGrade: grade.grade,
    reason: ''
  };
  applyDialogVisible.value = true;
};

// 提交申请
const submitApplication = async () => {
  if (!applyForm.value.newGrade || !applyForm.value.reason) {
    ElMessage.warning('请填写完整信息');
    return;
  }

  try {
    // 准备要提交的数据
    const gradeChange: Partial<GradeChangeDTO> = {
      gradeId: applyForm.value.gradeId,
      teacherId: teacherId.value,
      result: null,
      newGrade: applyForm.value.newGrade,
      reason: applyForm.value.reason,
      name: applyForm.value.name
    };

    // 发送申请
    await axios.post('/api/grade-changes', gradeChange);

    // 更新成绩状态
    await fetchGradeList();
    await fetchApplications();

    applyDialogVisible.value = false;
    ElMessage.success('申请已提交');
  } catch (error) {
    console.error('提交申请失败', error);
    ElMessage.error('提交失败，请重试');
  }
};

// 处理审核
const handleAudit = async (app: ApplicationDTO, result: boolean) => {
  ElMessageBox.confirm(`确定要${result ? '通过' : '拒绝'}该申请吗？`, '审核确认', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(async () => {
    try {
      // 发送审核结果
      await axios.put(`/api/grade-changes/${app.id}`, {
        gradeChangeId: parseInt(app.id),
        result: result,
        checkTime: new Date().toISOString()
      });

      // 更新列表
      await fetchGradeList();
      await fetchApplications();

      ElMessage.success(`已${result ? '通过' : '拒绝'}申请`);
    } catch (error) {
      console.error('审核失败', error);
      ElMessage.error('操作失败，请重试');
    }
  }).catch(() => {
    ElMessage.info('已取消操作');
  });
};
</script>
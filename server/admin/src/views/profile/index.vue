<script setup lang="ts">
import { reactive, ref } from "vue";
import { storeToRefs } from "pinia";
import { message } from "@/utils/message";
import { getErrorMessage } from "@/utils/error";
import { useUserStoreHook } from "@/store/modules/user";
import type { FormInstance, FormRules } from "element-plus";

defineOptions({
  name: "ProfileIndex"
});

const userStore = useUserStoreHook();
const { username, nickname } = storeToRefs(userStore);

// ---------- 编辑自己的显示名称 ----------
const profileFormRef = ref<FormInstance>();
const profileSaving = ref(false);
const profileForm = reactive({
  display_name: ""
});
// 预填当前显示名称（/me 已在登录/冷启动时加载）
profileForm.display_name = nickname.value ?? "";

const profileRules: FormRules = {
  display_name: [
    {
      validator: (_, value, callback) => {
        if (!String(value ?? "").trim()) {
          callback(new Error("显示名称不能为空"));
        } else {
          callback();
        }
      },
      trigger: "blur"
    }
  ]
};

/** 保存显示名称：调用 PATCH /me，成功后同步 Pinia（导航栏/本页立即更新） */
async function saveDisplayName() {
  if (!profileFormRef.value) return;
  await profileFormRef.value.validate(async valid => {
    if (!valid) return;
    const trimmed = profileForm.display_name.trim();
    if (trimmed === (nickname.value ?? "")) {
      message("显示名称未变化", { type: "info" });
      return;
    }
    profileSaving.value = true;
    try {
      await userStore.updateMyProfile(trimmed);
      message("显示名称已更新", { type: "success" });
    } catch (error) {
      // 失败保留原名称并显示一次中文错误
      profileForm.display_name = nickname.value ?? "";
      message(getErrorMessage(error), { type: "error" });
    } finally {
      profileSaving.value = false;
    }
  });
}
</script>

<template>
  <div class="page-fill">
    <!-- 个人资料面板 -->
    <div class="panel-container panel-fill">
      <div class="panel-header">
        <span class="panel-title">个人资料</span>
      </div>
      <div class="panel-body">
        <el-form
          ref="profileFormRef"
          :model="profileForm"
          :rules="profileRules"
          label-width="100px"
          style="max-width: 480px"
        >
          <el-form-item label="账号">
            <el-input :model-value="username || '—'" disabled />
          </el-form-item>
          <el-form-item label="显示名称" prop="display_name">
            <el-input
              v-model="profileForm.display_name"
              :placeholder="nickname || '显示名称'"
              clearable
            />
          </el-form-item>
          <el-form-item>
            <el-button
              type="primary"
              :loading="profileSaving"
              @click="saveDisplayName"
            >
              保存显示名称
            </el-button>
          </el-form-item>
        </el-form>
      </div>
    </div>
  </div>
</template>

<style scoped>
@import url("@/style/business.scss");
</style>

<script setup lang="ts">
import { reactive, ref, nextTick } from "vue";
import { useRouter } from "vue-router";
import { message } from "@/utils/message";
import { getErrorMessage } from "@/utils/error";
import { useUserStoreHook } from "@/store/modules/user";
import {
  createPasswordByteValidator,
  PASSWORD_PLACEHOLDER
} from "@/utils/password";
import { resetRouter } from "@/router";
import { useMultiTagsStoreHook } from "@/store/modules/multiTags";
import { routerArrays } from "@/layout/types";
import type { FormInstance, FormRules } from "element-plus";

defineOptions({
  name: "ChangePasswordDialog"
});

const router = useRouter();
const userStore = useUserStoreHook();

const visible = ref(false);
const passwordFormRef = ref<FormInstance>();
const passwordLoading = ref(false);
const passwordForm = reactive({
  current_password: "",
  new_password: "",
  confirm: ""
});

const passwordRules: FormRules = {
  current_password: [
    { required: true, message: "请输入当前密码", trigger: "blur" }
  ],
  new_password: [
    { required: true, message: "请输入新密码", trigger: "blur" },
    createPasswordByteValidator()
  ],
  confirm: [
    {
      validator: (_, value, callback) => {
        if (value !== passwordForm.new_password) {
          callback(new Error("两次输入的密码不一致"));
        } else {
          callback();
        }
      },
      trigger: "blur"
    }
  ]
};

/** 打开弹窗：重置表单与校验状态 */
function open() {
  visible.value = true;
  passwordForm.current_password = "";
  passwordForm.new_password = "";
  passwordForm.confirm = "";
  nextTick(() => {
    passwordFormRef.value?.clearValidate();
  });
}

/** 关闭后清空输入，避免下次打开残留 */
function handleClosed() {
  passwordForm.current_password = "";
  passwordForm.new_password = "";
  passwordForm.confirm = "";
  passwordFormRef.value?.clearValidate();
}

/** 修改密码成功后后端吊销当前会话，需要重新登录 */
async function submitPassword() {
  if (!passwordFormRef.value) return;
  await passwordFormRef.value.validate(async valid => {
    if (!valid) return;
    passwordLoading.value = true;
    try {
      await userStore.changePassword({
        current_password: passwordForm.current_password,
        new_password: passwordForm.new_password
      });
      message("密码已修改，请重新登录", { type: "success" });
      visible.value = false;
      userStore.resetAuthState();
      useMultiTagsStoreHook().handleTags("equal", [...routerArrays]);
      resetRouter();
      router.push("/login");
    } catch (error) {
      message(getErrorMessage(error), { type: "error" });
    } finally {
      passwordLoading.value = false;
    }
  });
}

defineExpose({ open });
</script>

<template>
  <el-dialog
    v-model="visible"
    title="修改密码"
    width="480px"
    destroy-on-close
    @closed="handleClosed"
  >
    <el-alert
      type="info"
      :closable="false"
      class="mb-3"
      title="修改成功后当前会话将被吊销，需要重新登录。"
    />
    <el-form
      ref="passwordFormRef"
      :model="passwordForm"
      :rules="passwordRules"
      label-width="100px"
    >
      <el-form-item label="当前密码" prop="current_password">
        <el-input
          v-model="passwordForm.current_password"
          type="password"
          show-password
          placeholder="当前登录密码"
        />
      </el-form-item>
      <el-form-item label="新密码" prop="new_password">
        <el-input
          v-model="passwordForm.new_password"
          type="password"
          show-password
          :placeholder="PASSWORD_PLACEHOLDER"
        />
      </el-form-item>
      <el-form-item label="确认新密码" prop="confirm">
        <el-input
          v-model="passwordForm.confirm"
          type="password"
          show-password
          placeholder="再次输入新密码"
        />
      </el-form-item>
    </el-form>
    <template #footer>
      <el-button @click="visible = false">取消</el-button>
      <el-button
        type="primary"
        :loading="passwordLoading"
        @click="submitPassword"
      >
        确认修改
      </el-button>
    </template>
  </el-dialog>
</template>

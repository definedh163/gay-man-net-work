<template>
  <div>
    <div
      style="
        margin-bottom: 10px;
        margin-right: 10px;
        display: flex;
        justify-content: flex-end;
      "
    >
      <a-button @click="sendDataToBackend" type="dashed">上传</a-button>
    </div>

    <!-- <input type="file" @change="handleFileUpload" /> -->

    <a-table
      :dataSource="dataSource"
      :columns="columns"
      :loading="loading"
      row-key="id"
      :pagination="pagination"
      @change="handleTableChange"
    >
      <template #bodyCell="{ column, record }">
        <template v-if="column.dataIndex === 'operation'">
          <div style="display: flex; justify-content: space-between">
            <a-button
              style="color: green"
              type="link"
              @click="onEdit(record.id)"
            >
              <template #icon> </template>编辑</a-button
            >
            <a-button
              v-if="record.files.downloadUrl"
              style="color: #0958d9"
              type="link"
              :href="record.files.downloadUrl"
            >
              <template #icon> </template>下载</a-button
            >

            <a-button
              style="color: red"
              type="link"
              @click="onDelete(record.id)"
            >
              <template #icon> </template>删除</a-button
            >
          </div>
        </template>
        <template v-if="column.dataIndex === 'fileName'">
          <span>{{ record.files.fileName }}</span>
        </template>
        <template v-if="column.dataIndex === 'fileSize'">
          <span>{{ record.files.fileSize }}</span>
        </template>
      </template>
    </a-table>

    <formModal ref="formModalRef" @close="onClose"></formModal>
  </div>
</template>

<script setup>
import axios from "axios";
import formModal from "./form-modal.vue";
import { onMounted, reactive, ref, unref, computed } from "vue";
import { message } from "ant-design-vue";

// 后端地址
const apiUrl = "http://localhost:3000"; // 替换为你的后端 API 地址

const dataSource = ref([]);
const formModalRef = ref(undefined);
const searchForm = reactive({
  current: 1,
  pageSize: 10,
});
const total = ref(0);
const loading = ref(false);

const columns = [
  {
    title: "序号",
    dataIndex: "index",
    align: "center",
    width: 100,
    customRender: ({ renderIndex }) => {
      return (searchForm.current - 1) * 10 + renderIndex + 1;
    },
  },
  {
    title: "文件别名",
    dataIndex: "name",
    key: "text",
  },
  {
    title: "文件名称",
    dataIndex: "fileName",
  },
  {
    title: "文件大小",
    dataIndex: "fileSize",
  },
  {
    title: "操作",
    dataIndex: "operation",
    align: "center",
    width: 100,
    fixed: "right",
  },
];

onMounted(() => {
  getDataSource();
});

const pagination = computed(() => ({
  total: total.value,
  current: searchForm.pageNo,
  pageSize: searchForm.pageSize,
  showSizeChanger: true, // 是否显示pagesize选择
  showQuickJumper: true, // 是否显示跳转窗
  showTotal: (total) => `共 ${total} 条`,
}));

const sendDataToBackend = () => {
  formModalRef.value.onShow();
};

const getDataSource = () => {
  const url = `${apiUrl}/get-table-data`;
  loading.value = true;
  axios
    .get(url, {
      params: unref(searchForm),
    })
    .then((res) => {
      let { code, data } = res.data;
      if (code === 200) {
        dataSource.value = data
          .map((el) => {
            if (el.files) {
              el.files = JSON.parse(el.files);
            }
            return el;
          })
          .filter((f) => f != undefined);
        total.value = res.data.total;
        loading.value = false;
      }
    })
    .catch((error) => {
      console.error("获取数据失败:", error);
    });
};

const handleTableChange = (pag) => {
  searchForm.current = pag.current;
  searchForm.pageSize = pag.pageSize;
  getDataSource();
};

const onDelete = async (id) => {
  const deleteUrl = `${apiUrl}/delete-table/${id}`;

  axios
    .delete(deleteUrl)
    .then((res) => {
      if (res.data.code == 200) {
        message.success(res.data.msg);
        getDataSource();
      } else {
        message.error("删除数据失败!");
      }
    })
    .catch(() => {
      message.error("系统异常!");
    });
};

const onEdit = (id) => {
  console.log("🚀 ~ onEdit ~ id:", id);
  formModalRef.value.onShow(id);
};

const onClose = () => {
  getDataSource();
};
</script>

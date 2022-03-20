<template>
  <el-dialog v-model="exampleDialogVisible" title="表格输入样例" width="90%">
    <p>用|划分，第一行默认为表头，后面是对应的表格值，比如：</p>
    <p>
      科目|时间<br />
      语文|30 分钟作业
    </p>
    <template #footer>
      <span class="dialog-footer">
        <el-button type="primary" @click="exampleDialogVisible = false"
          >确认</el-button
        >
      </span>
    </template>
  </el-dialog>

  <el-dialog v-model="usageDialogVisible" title="网站使用说明" width="90%">
    <p>首先可以设置起床时间、睡眠时间、入睡时间</p>
    <p>三个开关分别是：是否开启暗色模式、是否显示序号、表格编辑器切换</p>
    <hr />
    <p>插入功能：</p>
    <p>科目输入需要插入的序号，内容输入插入到的序号前，然后点击插入</p>
    <p>
      比如：当我有一个最大序号为 5 的表格，在科目输入 4，内容输入
      2，则第四行内容将会插入到第二行，即第四行内容变为第二行，后面序号依次顺延
    </p>
    <hr />
    <p>最后那个表格前表格后文字即在表格前和表格后显示的文字。支持 HTML</p>
    <hr />
    <p>生成图片的时候图片宽度固定</p>
    <template #footer>
      <span class="dialog-footer">
        <el-button type="primary" @click="usageDialogVisible = false"
          >确认</el-button
        >
      </span>
    </template>
  </el-dialog>

  <div class="control">
    <div>
      <!-- 是否为暗色模式 -->
      <el-switch
        v-model="isDark"
        inline-prompt
        active-text="暗"
        inactive-text="亮"
        active-color="#1b1f2e"
      /><br />
      <!-- 是否开启序号 -->
      <el-switch
        v-model="needOrderNumber"
        inline-prompt
        active-text="序"
        inactive-text="无序"
      /><br />
      <!-- 是否使用类似 markdown 的语法渲染表格 -->
      <el-switch
        v-model="allInOneEditor"
        inline-prompt
        active-text="合"
        inactive-text="分"
      /><br />
      <el-button @click="usageDialogVisible = true">说明</el-button>
      <el-button @click="generateImage">生成图片</el-button>
    </div>

    <div class="table-setting">
      <!-- 一行一行添加表格 -->
      <el-form :model="table" @submit.prevent v-if="!allInOneEditor">
        <el-form-item label="科目：">
          <el-input v-model="table.subject" name="wakeUpTime" />
        </el-form-item>
        <el-form-item label="内容：">
          <el-input v-model="table.subjectTime" name="lunchRestTime" />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="addSubject">添加</el-button>
          <el-button type="danger" @click="removeSubject">删除</el-button>
          <el-button @click="insertSubject">插入</el-button>
        </el-form-item>
      </el-form>

      <!-- 类似 markdown 语法添加表格 -->
      <el-form :model="table" @submit.prevent v-if="allInOneEditor">
        <el-form-item>
          <el-input v-model="table.allInOne" type="textarea" />
        </el-form-item>
        <el-form-item>
          <el-button @click="exampleDialogVisible = true">示例</el-button>
        </el-form-item>
      </el-form>
    </div>

    <div class="other-text-setting">
      <!-- 其他文字设置 -->
      <el-form :model="otherText">
        <el-form-item label="表格前文字：">
          <el-input v-model="otherText.before" type="textarea" />
        </el-form-item>
        <el-form-item label="表格后文字：">
          <el-input v-model="otherText.after" type="textarea" />
        </el-form-item>
      </el-form>
    </div>
  </div>

  <div class="render" :class="isDark ? 'black' : 'light'" id="render">
    <p
      v-for="(value, index) of otherTextShow.before"
      :key="index"
      v-html="value"
    ></p>

    <!-- 一条一条插入式时使用的表格 -->
    <table v-if="!allInOneEditor">
      <thead>
        <tr>
          <th v-if="needOrderNumber">序号</th>
          <th>学科</th>
          <th>内容</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(value, index) of subjectsTimes" :key="index">
          <th v-if="needOrderNumber">{{ index + 1 }}</th>
          <th>{{ value.subject }}</th>
          <th>{{ value.content }}</th>
        </tr>
      </tbody>
    </table>

    <!-- 类似 markdown 直接解析的表格 -->
    <table v-if="allInOneEditor">
      <thead>
        <tr>
          <th v-for="(value, index) of tableAllInOne.title" :key="index">
            {{ value }}
          </th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(value, index) of tableAllInOne.content" :key="index">
          <th v-for="(val, i) of value" :key="`${index}-${i}`">{{ val }}</th>
        </tr>
      </tbody>
    </table>

    <p
      v-for="(value, index) of otherTextShow.after"
      :key="index"
      v-html="value"
    ></p>
  </div>
  <footer>
    This project is open-source on
    <a href="https://github.com/Rotten-LKZ/table-generator" target="_blank"
      >GitHub</a
    >
    and
    <a
      href="https://gitee.com/rotten_lkz/article-image-generator"
      target="_blank"
      >Gitee</a
    >
  </footer>
</template>

<script lang="ts">
import { ElMessage } from "element-plus";
import { computed, defineComponent, reactive, ref } from "vue";
import { toJpeg } from "html-to-image";
import download from "downloadjs";

interface SubjectsTimes {
  subject: string;
  content: string;
}

interface OtherTextShow {
  before: string[];
  after: string[];
}

export default defineComponent({
  name: "App",
  setup() {
    // 表格具体内容变量（给表单的变量）
    const table = reactive({
      subject: "",
      subjectTime: "",
      allInOne: `科目|内容
语文|
数学|
英语|
历史|
道法|
物理|
化学|
体育|`,
    });

    // 其他文字变量（给表单的变量）
    const otherText = reactive({
      before: `起床时间：<code>7:00</code>
午睡时间：<code>30 分钟</code>`,
      after: `入睡时间：<code>7:00</code>`,
    });

    // 其他文字显示变量（渲染时用的变量）
    const otherTextShow = computed((): OtherTextShow => {
      return {
        before: otherText.before.trim().split("\n"),
        after: otherText.after.trim().split("\n"),
      };
    });

    // 给表格用的变量（渲染时用的变量）
    const tableAllInOne = computed(() => {
      const lines = table.allInOne.trim().split("\n");
      let res: string[][] = [];
      for (let i = 0; i < lines.length; i++) {
        res.push(lines[i].split("|"));
        if (needOrderNumber.value) {
          if (i === 0) {
            // 表头
            res[i].unshift("序号");
          } else {
            // 表体
            res[i].unshift(i.toString());
          }
        }
      }
      return {
        title: res[0],
        content: res.slice(1),
      };
    });

    // 表格变量
    const subjectsTimes = ref(new Array<SubjectsTimes>());

    // 是否为暗色模式
    const isDark = ref(false);
    // 是否用类似 markdown 解析
    const allInOneEditor = ref(false);
    // 是否打开示例 dialog
    const exampleDialogVisible = ref(false);
    // 是否打开网站使用说明 dialog
    const usageDialogVisible = ref(false);
    // 是否显示序号
    const needOrderNumber = ref(false);

    // 添加一行表格
    function addSubject() {
      subjectsTimes.value.push({
        subject: table.subject,
        content: table.subjectTime,
      });
      table.subject = "";
      table.subjectTime = "";
    }

    // 删除一行表格
    function removeSubject() {
      subjectsTimes.value = subjectsTimes.value.filter(
        (val) => val.subject !== table.subject
      );
      table.subject = "";
      table.subjectTime = "";
    }

    // 插入
    function insertSubject() {
      const oldIndex = parseInt(table.subject) - 1;
      const newIndex = parseInt(table.subjectTime) - 1;
      if (
        oldIndex < 0 ||
        newIndex < 0 ||
        isNaN(oldIndex) ||
        isNaN(newIndex) ||
        oldIndex >= subjectsTimes.value.length ||
        newIndex >= subjectsTimes.value.length
      ) {
        ElMessage.error("请输入正确的序号");
        return;
      }

      const content = subjectsTimes.value[oldIndex];
      let newArr = subjectsTimes.value;
      newArr.splice(oldIndex, 1);
      newArr.splice(newIndex, 0, content);

      subjectsTimes.value = newArr;

      table.subject = "";
      table.subjectTime = "";
    }

    // 生成图片
    function generateImage() {
      const node = document.getElementById("render");
      if (node === null) {
        ElMessage.error("没有找到渲染节点");
        return;
      }
      node.classList.add("generation");
      toJpeg(node)
        .then(function (dataUrl) {
          download(dataUrl, "table.png");
          node.classList.remove("generation");
        })
        .catch(function (error) {
          ElMessage.error("生成失败，可重试或者尝试截图" + error);
        });
    }

    return {
      table,
      otherText,
      otherTextShow,
      subjectsTimes,
      isDark,
      allInOneEditor,
      tableAllInOne,
      exampleDialogVisible,
      usageDialogVisible,
      needOrderNumber,

      addSubject,
      removeSubject,
      insertSubject,
      generateImage,
    };
  },
});
</script>

<style>
body {
  padding: 0;
  margin: 0;
  overflow-x: hidden;
}

code {
  padding: 3px 6px;
  font-size: 0.85rem;
  border-radius: 3px;
  font-family: "Source Code Pro", Consolas, Monaco, SFMono-Regular,
    "Ubuntu Mono", Menlo, monospace;
}

table {
  width: 100%;
  border-collapse: collapse;
  box-sizing: border-box;
  text-align: center;
  font-size: 1rem;
  line-height: 1.8;
  overflow-wrap: break-word;
}

table thead th {
  font-weight: 600;
}

table tbody th {
  font-weight: 400;
}

table th {
  padding: 6px 13px;
  border: 1px solid #4eaaff;
}

.control {
  margin-top: 10px;
  padding: 10px;
  gap: 10px;
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: center;
  width: 100%;
  box-sizing: border-box;
}

.control textarea {
  flex: 1 0 auto;
  resize: horizontal;
  height: 150px;
}

.render {
  margin: 30px auto;
  width: 100%;
  max-width: 800px;
  padding: 20px;
  box-sizing: border-box;
  font-weight: 400;
  font-family: "PingFang SC", "Microsoft YaHei", Roboto, Arial, sans-serif;
}

.generation {
  margin: 0 !important;
  width: 800px !important;
}

.black {
  background: #1b1f2e;
  color: #f2f2f2;
}

.black code {
  color: #eee;
  background: #303030;
}

.black table tbody tr:hover {
  background: #303030;
}

.light {
  background: #ffffff;
  color: #333;
}
.light code {
  color: #555;
  background-color: #f5f5f5;
}

.light table tbody tr:hover {
  background: #f5f5f5;
}

footer {
  text-align: center;
  color: #7a7a7ab4;
  margin-top: 10px;
  padding-bottom: 20px;
}
</style>

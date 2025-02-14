<template>
  <div class="rule">
    <div class="radius-div">
      <el-table
        :data="ruleList"
        :default-sort="{prop: 'alias'}"
        style="margin: 20px">
        <el-table-column
          prop="id"
          label="ID"
          width="100">
        </el-table-column>
        <el-table-column
          sortable
          prop="alias"
          label="别名">
        </el-table-column>
        <el-table-column
          sortable
          prop="fitTime"
          label="持续时间">
        </el-table-column>
        <el-table-column
          sortable
          :sort-method="(a, b) => +a.priority - +b.priority"
          prop="priority"
          label="优先级">
        </el-table-column>
        <el-table-column
          sortable
          prop="type"
          label="类型">
        </el-table-column>
        <el-table-column
          label="操作"
          width="256">
          <template slot-scope="scope">
            <el-button
              type="primary"
              size="small"
              v-clipboard:copy="JSON.stringify(scope.row, null, 2)"
              v-clipboard:success="onCopy"
              v-clipboard:error="onError">
              复制
            </el-button>
            <el-button @click="modifyRule(scope.row)" type="warning" size="small">编辑</el-button>
            <el-button @click="deleteRule(scope.row)" :disabled="scope.row.used" type="danger" size="small">删除</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="radius-div">
      <el-collapse  class="collapse" v-model="ruleCollapse">
        <el-collapse-item title="新增 | 编辑 删种规则" name="1">
          <div style="width: fit-content; margin: 6px 0 12px 20px">
            <el-tag size="small">规则 ID: {{rule.id || '新增'}}</el-tag>
          </div>
          <div style="width: fit-content; margin: 6px 0 12px 20px">
            <el-tag size="small" type="warning">以下条件必须全部符合, 才触发删除种子操作, 留空为不启用选项, 优先删除已经完成时间早的种子, 若种子未完成, 则优先删除添加早的种子</el-tag>
          </div>
          <el-form ref="rule" class="rule-form" :model="rule" label-width="160px" size="mini">
            <el-form-item required label="别名" prop="alias">
              <el-input v-model="rule.alias" type="input"></el-input>
            </el-form-item>
            <el-form-item label="持续时间" prop="fitTime">
              <el-input v-model="rule.fitTime" type="input"></el-input>
              <div><el-tag type="info">符合删种规则的持续时间, 只有到达持续时间之后才会删种, 单位为 秒, 不启用留空, 建议考虑下载器的删种周期一起设置</el-tag></div>
            </el-form-item>
            <el-form-item label="优先级" prop="priority">
              <el-input v-model="rule.priority" type="input"></el-input>
              <div><el-tag type="info">优先级越高的规则越执行, 默认为 0</el-tag></div>
            </el-form-item>
            <el-form-item label="单次删种数量" prop="deleteNum">
              <el-input v-model="rule.deleteNum" type="input"></el-input>
              <div><el-tag type="info">单次删种任务的删除种子的数量, 不同规则间不累计, 留空为单次只删除一个</el-tag></div>
            </el-form-item>
            <el-form-item required label="类型" prop="type">
              <el-select v-model="rule.type" style="width: 144px" placeholder="类型">
                  <el-option label="普通" value="normal"></el-option>
                  <el-option label="JavaScript" value="javascript"></el-option>
                </el-select>
            </el-form-item>
            <el-form-item v-if="rule.type === 'normal'" label="限制条件">
              <el-table
                size="mini"
                :data="rule.conditions"
                style="width: 720px">
                <el-table-column
                  label="选项"
                  width="180">
                  <template slot-scope="scope">
                    <el-select v-model="scope.row.key" style="width: 160px" placeholder="选择选项">
                      <el-option v-for="item of conditionKeys" :key="item.key" :label="item.name" :value="item.key"></el-option>
                    </el-select>
                  </template>
                </el-table-column>
                <el-table-column
                  label="比较类型"
                  width="144">
                  <template slot-scope="scope">
                    <el-select v-model="scope.row.compareType" style="width: 120px" placeholder="比较类型">
                      <el-option label="等于" value="equals"></el-option>
                      <el-option label="大于" value="bigger"></el-option>
                      <el-option label="小于" value="smaller"></el-option>
                      <el-option label="包含" value="contain"></el-option>
                      <el-option label="包含于" value="includeIn"></el-option>
                      <el-option label="不包含" value="notContain"></el-option>
                      <el-option label="不包含于" value="notIncludeIn"></el-option>
                    </el-select>
                  </template>
                </el-table-column>
                <el-table-column
                  label="值">
                  <template slot-scope="scope">
                    <el-input v-model="scope.row.value" placeholder="填写值"/>
                  </template>
                </el-table-column>
                <el-table-column
                  label="操作"
                  width="96">
                  <template slot-scope="scope">
                    <el-button @click="rule.conditions = rule.conditions.filter(item => item !== scope.row)" type="danger" size="small">删除</el-button>
                  </template>
                </el-table-column>
              </el-table>
              <el-button @click="rule.conditions.push({ ...condition })" type="primary" size="small">新增</el-button>
              <el-card style="margin: 12px 24px 12px 0;" >
                说明: <br>
                01. 分享率一: 上传 / 种子大小 的结果<br>
                02. 分享率二: 上传 / 下载 的结果<br>
                03. 站点域名: 种子的 Tracker 地址的域名部分<br>
                04. 各类时间: 选项时间到当前时间的差值, 单位为 秒/s<br>
                05. 各类大小: 单位为 字节 / Byte, 可以使用 * 做乘法运算<br>
                06. 各类速度: 单位为 字节/s / Byte/s<br>
                07. 种子状态: 参照 qBittorrent 对种子状态的定义, 主要包含以下几类: <br>
                &nbsp;&nbsp;&nbsp;&nbsp;上传中: <el-tag size="mini">uploading</el-tag>, 下载中: <el-tag size="mini">downloading</el-tag><br>
                &nbsp;&nbsp;&nbsp;&nbsp;等待下载: <el-tag size="mini">stalledDL</el-tag>, 做种但无上传: <el-tag size="mini">stalledUP</el-tag><br>
                &nbsp;&nbsp;&nbsp;&nbsp;更多状态请参照 qBittorrent Wiki, 若想删除等待下载状态下的种子, 应填写 <el-tag size="mini">stalledDL</el-tag><br>
                08. 返回信息: 种子 Tracker 列表中由 Tracker 返回的信息<br>
                09. 当前时间: 当天 0 点到当前时间的秒数<br>
                &nbsp;&nbsp;&nbsp;&nbsp;例: 填写 当前时间大于 8*3600 与 当前时间小于 22*3600<br>
                &nbsp;&nbsp;&nbsp;&nbsp;则只会在当天上午 8 点之后到 22 点之前删种<br>
                &nbsp;&nbsp;&nbsp;&nbsp;0 点的时间戳取决于 Vertex 安装环境的时区<br>
                10. 全局速度: 当前下载器的速度<br>
                11. 做种下载连接: 仅计算已连接上的数量, 也即 qBittorrent WebUI 内括号外的数字 <br>
                12. 做种下载任务: 任务的数量, 做种包含上传中状态与做种状态, 下载包含下载中与等待下载状态 <br>
                13. 比较类型中的 包含 / 包含于 或 不包含 / 不包含于: 值部分需以半角逗号 , 为分割符, 如种子分类不包含于 KEEP, KEEP2, KEEP3 三个分类, 则应填写:
                <el-tag size="mini">KEEP,KEEP2,KEEP3</el-tag><br>
              </el-card>
            </el-form-item>
            <el-form-item v-if="rule.type === 'javascript'" label="自定义代码">
              <el-input v-model="rule.code" type="textarea" :rows="20" style="width: 500px;"></el-input>
              <div><el-tag type="info">自定义删种逻辑代码</el-tag></div>
            </el-form-item>
            <el-form-item size="small">
              <el-button type="primary" @click="handleRuleClick">新增 | 编辑</el-button>
              <el-button type="primary" @click="importRuleVisible = !importRuleVisible">导入</el-button>
              <el-button @click="clearRule">清空</el-button>
            </el-form-item>
          </el-form>
        </el-collapse-item>
      </el-collapse>
    </div>
    <el-dialog title="导入规则" :visible.sync="importRuleVisible" width="80%">
      <el-form label-width="144px" size="mini" style="width: 80%;">
        <el-form-item label="规则">
          <el-input v-model="importRuleText" type="textarea" :rows="20"  style="width: 500px;"></el-input>
        </el-form-item>
        <el-form-item size="mini">
          <el-button type="primary" @click="importRule">导入</el-button>
        </el-form-item>
      </el-form>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data () {
    return {
      rule: {},
      conditionKeys: [{
        name: '种子名称',
        key: 'name'
      }, {
        name: '种子进度',
        key: 'progress'
      }, {
        name: '上传速度',
        key: 'uploadSpeed'
      }, {
        name: '下载速度',
        key: 'downloadSpeed'
      }, {
        name: '种子分类',
        key: 'category'
      }, {
        name: '种子标签',
        key: 'tags'
      }, {
        name: '种子大小',
        key: 'size'
      }, {
        name: '种子状态',
        key: 'state'
      }, {
        name: '站点域名',
        key: 'tracker'
      }, {
        name: '返回信息',
        key: 'trackerStatus'
      }, {
        name: '已完成量',
        key: 'completed'
      }, {
        name: '已下载量',
        key: 'downloaded'
      }, {
        name: '已上传量',
        key: 'uploaded'
      }, {
        name: '分享率一',
        key: 'ratio'
      }, {
        name: '分享率二',
        key: 'trueRatio'
      }, {
        name: '添加时间',
        key: 'addedTime'
      }, {
        name: '完成时间',
        key: 'completedTime'
      }, {
        name: '保存路径',
        key: 'savePath'
      }, {
        name: '做种连接',
        key: 'seeder'
      }, {
        name: '下载连接',
        key: 'leecher'
      }, {
        name: '剩余空间',
        key: 'freeSpace'
      }, {
        name: '下载任务',
        key: 'leechingCount'
      }, {
        name: '做种任务',
        key: 'seedingCount'
      }, {
        name: '全局上传',
        key: 'globalUploadSpeed'
      }, {
        name: '全局下载',
        key: 'globalDownloadSpeed'
      }, {
        name: '当前时间',
        key: 'secondFromZero'
      }],
      defaultRule: {
        conditions: [],
        priority: 0,
        code: '(maindata, torrent) => {\n' +
              '  return false;\n' +
              '}'
      },
      condition: {
        key: '',
        compareType: '',
        value: ''
      },
      ruleList: [],
      ruleCollapse: ['1'],
      importRuleVisible: false,
      importRuleText: ''
    };
  },
  methods: {
    async handleRuleClick () {
      this.$refs.rule.validate(async (valid) => {
        if (valid) {
          const url = '/api/deleteRule/' + (this.rule.id ? 'modify' : 'add');
          const res = await this.$axiosPost(url, this.rule);
          if (!res) {
            return;
          }
          await this.$messageBox(res);
          this.listRule();
          this.clearRule();
        } else {
          return false;
        }
      });
    },
    async deleteRule (row) {
      const url = '/api/deleteRule/delete';
      const res = await this.$axiosPost(url, {
        id: row.id
      });
      if (!res) {
        return;
      }
      await this.$messageBox(res);
      this.listRule();
      this.clearRule();
    },
    async modifyRule (row) {
      this.ruleCollapse = ['1'];
      this.rule = { ...row };
      this.rule.conditions = row.conditions.map(item => {
        return { ...item };
      });
    },
    async clearRule () {
      this.rule = { ...this.defaultRule };
      this.rule.conditions = [{ ...this.condition }];
      this.$refs.rule.resetFields();
    },
    async listRule () {
      const res = await this.$axiosGet('/api/deleteRule/list');
      this.ruleList = res ? res.data : [];
    },
    onCopy () {
      this.$message.info('复制到剪贴板成功');
    },
    onError (e) {
      this.$message.error(e.message);
    },
    importRule () {
      this.clearRule();
      try {
        this.rule = {
          ...JSON.parse(this.importRuleText),
          id: undefined
        };
        this.importRuleVisible = false;
        this.importRuleText = '';
      } catch (e) {
        this.$message.error('输入内容有误');
      }
    }
  },
  async mounted () {
    this.rule = { ...this.defaultRule };
    this.rule.conditions = [{ ...this.condition }];
    this.$refs.rule.resetFields();
    this.listRule();
  }
};
</script>

<style scoped>
.rule-div {
  margin: 20px 0;
}

.collapse {
  margin: 20px;
}

.rule-form * {
  width: fit-content;
  text-align: left;
}
</style>

<template>
    <div>
        <div class="template-content" v-loading="loading" element-loading-text="请稍等···" :style="{ 'height': temPanelHeight }">
            <el-row>
                <div class="pull-left template-title">
                    当前{{this.templateType}}：<span class="template-name">{{templateInfo.templateName}}</span>
                    <el-tooltip v-if="templateInfo.content === '' || templateInfo.content === '{}'" effect="dark" content="初始化自定义业务模板" placement="bottom">
                      <el-button type="success" @click="setCustomTemplate" size="small" v-if="isEditOperate && !templateInfo.approving">
                        <i class="fa fa-hand-o-right"></i>模板向导
                      </el-button>
                    </el-tooltip>
                    <el-tooltip v-if="templateInfo.content !== '' && templateInfo.content !== '{}'" effect="dark" content="可在当前光标处，插入选择的模板字段" placement="bottom">
                      <el-button type="primary" @click="insertTemplate" size="small" v-if="isEditOperate && !templateInfo.approving">
                        <i class="fa fa-code"></i>插入查询变量
                      </el-button>
                    </el-tooltip>
                    <span v-if="templateInfo.approving && templateInfo.type === 1" class="template-approving">状态：<router-link tag="a" :to="{ name: 'authority_manage' }">待审核</router-link>，不可进行保存，删除等操作</span>
                    <span v-if="templateInfo.approving && templateInfo.type === 0" class="template-approving">引用该宏的模板处于待审核状态，不可进行保存、删除等操作</span>
                </div>
                <div class="pull-right" v-if="isAllPrivilege">
                    <log-monitor :template-name="templateInfo.templateName" :index-id="indexId" :index-name="indexName"></log-monitor>
                    <el-select v-if="!isMacroVisible && isEditOperate" size="small" placeholder="请选择要插入的宏" v-model="selectedMacro" style="padding-right: 10px;" clearable @change="insertMacro">
                        <el-option v-for="item in macroList" :label="item.templateName" :value="item.templateName" :key="item.templateName"></el-option>
                    </el-select>
                    <el-button type="primary" @click="handleSave" size="small" v-if="isEditOperate && !templateInfo.approving">保存</el-button>
                    <el-button type="primary" @click="handleApprove" size="small" v-if="isEditOperate && !templateInfo.approving && templateInfo.type === 1">提交</el-button>
                    <el-button type="danger" @click="handleDelete" size="small" v-if="isEditOperate && !templateInfo.approving">删除</el-button>
                    <el-button type="primary" @click="handleHistoryVersion" size="small" v-if="templateInfo.hisCount > 0 && isEditOperate">{{historyVersionBtn}}</el-button>
                </div>
            </el-row>
            <div>
                <el-row>
                    <el-tabs v-model="activeTab" @tab-click="tabClick">
                        <el-tab-pane label="编辑模板" name="edit">
                            <div :class="[isShowHistoryVersion ? 'template-edit-and-version-content' : 'template-edit-content']" :style="{ 'height': temPanelHeight - 85 }">
                                <editor ref="aceEditor1" :content="templateInfo.content" v-on:change-content="changeEditContent" :editor-id="eidtorId"></editor>
                            </div>
                            <div v-show="isShowHistoryVersion" class="template-history-version-content">
                                <div style="padding-left:10px;">
                                    <el-table :data="historyVersionList" border @row-click="handleVersionDialog" :max-height="temPanelHeight - 85">
                                        <el-table-column label="修改日期" width="150px">
                                              <template slot-scope="scope">{{scope.row.createdTime | formatDate}}</template>
                                        </el-table-column>
                                        <el-table-column prop="description" label="描述">
                                        </el-table-column>
                                        <el-table-column prop="creator" label="修改者"></el-table-column>
                                    </el-table>
                                </div>
                            </div>
                        </el-tab-pane>
                        <el-tab-pane label="sql parser" name="sql" :disabled="isMacroVisible || !isAllPrivilege">
                            <div style="margin: 5px 0 10px;">
                                <span>数据源：</span>
                                <el-select size="medium" v-model="datasourceId" placeholder="请选择数据源" style="width: 39%;" @change="initSql">
                                    <el-option v-for="item in Object.entries(datasourceList)" :key="item[0]" :label="item[1]" :value="item[0]"></el-option>
                                </el-select>
                            </div>
                            <el-col :span="11">
                                <div :style="{ 'height': temPanelHeight - 135 }">
                                    <editor ref="sqlEditor" :content="sql" v-on:change-content="changeSqlContent" mode="sql" editor-id="sqlEditor"></editor>
                                </div>
                            </el-col>
                            <el-col :span="2">
                              <div :style="{'margin-top': (temPanelHeight - 240) / 2}" align="center">
                                  <div><el-button size="small" style="width: 75px;" title="结果仅供参考，需进一步加工" type="primary" @click="handleExplain">转 DSL</el-button></div>
                                  <div style="margin-top: 5px;"><el-button title="谨慎执行，别跑挂DB了" :disabled="!isAllPrivilege" style="width: 75px;" size="small" type="primary" @click="handleExecute">查询DB</el-button></div>
                              </div>
                            </el-col>
                            <el-col :span="11">
                              <div :style="{ 'height': temPanelHeight - 135 }">
                                  <editor ref="sqlEditorResult" :content="explainContent" :readonly="true" editor-id="sqlEditorResult"></editor>
                              </div>
                            </el-col>
                        </el-tab-pane>
                        <el-tab-pane label="调试" name="debug" :disabled="isMacroVisible || !isAllPrivilege">
                              <div class="render-cluster" v-if="clusters.length > 1">
                                  <span>指定集群：</span>
                                  <el-select size="small" v-model="clusterId" placeholder="请选择集群" style="margin-left: 10px;">
                                      <el-option v-for="item in clusters" :key="item.id" :label="item.clusterId" :value="item.id"></el-option>
                                  </el-select>
                              </div>
                                <el-col :span="11">
                                    <div class="debug-title pull-left">
                                      <div class="pull-left" style="margin-right: 10px;">参数</div>
                                      <div class="pull-right">
                                        <el-button size="small" type="primary" @click="handleResetParams">重置参数</el-button>
                                        <el-button size="small" type="primary" @click="handleFormatParams">格式化参数</el-button>
                                      </div>
                                    </div>
                                    <div :style="{ 'height': temPanelHeight - 120 }">
                                        <editor ref="aceEditor2" :content="templateInfo.params" v-on:change-content="changeDebugContent" :editor-id="debugId"></editor>
                                    </div>
                                </el-col>
                                <el-col :span="2">
                                  <div :style="{'margin-top': (temPanelHeight - 240) / 2}" align="center">
                                      <div><el-button size="mini" style="width: 75px;" type="primary" @click="handleRender">渲染DSL</el-button></div>
                                      <div style="margin-top: 10px;"><el-button style="width: 75px;" size="mini" type="primary" @click="handleDebug">执行查询</el-button></div>
                                      <div style="margin-top: 10px;"><el-button style="width: 75px;" size="mini" type="primary" @click="queryProfile">慢查询分析</el-button></div>
                                  </div>
                                </el-col>
                                <el-col :span="11">
                                  <div>
                                    <div class="debug-title">
                                      <span>结果</span>
                                      <el-button
                                      v-if="!isProfileVisible"
                                      type="primary"
                                      size="small"
                                      class="pull-right"
                                      v-clipboard:copy="resultContent"
                                      v-clipboard:success="copySuccess"
                                      v-clipboard:error="copyError">复制内容
                                    </el-button>
                                    </div>
                                    <div :style="{ 'height': temPanelHeight - 120 }">
                                        <editor v-if="!isProfileVisible" ref="debugResultEditor" :content="resultContent" :readonly="true" editor-id="debugResult"></editor>
                                        <profile-content v-else :profile-data="profileData"></profile-content>
                                    </div>
                                  </div>
                                </el-col>
                        </el-tab-pane>
                        <el-tab-pane label="API" name="api" :disabled="isMacroVisible || !isAllPrivilege">
                            <div class="api-content" :style="{ 'height': temPanelHeight - 95 }">
                                <el-scrollbar>
                                    RestClient示例:
                                    <br/>
                                    <pre>{{apiContent.rest_client}}</pre>
                                    <br/>
                                    <pre>{{`POST: ${apiContent.path}`}}</pre>
                                    <pre>{{apiContent.content}}</pre>
                                </el-scrollbar>
                            </div>
                        </el-tab-pane>
                        <el-tab-pane label="性能测试" name="test" :disabled="isMacroVisible || !isAllPrivilege">
                            <template-test :index-id="indexId" :template-name="templateInfo.templateName" :params-info="paramsInfo" :tem-panel-height="temPanelHeight"></template-test>
                        </el-tab-pane>
                        <el-tab-pane label="超时重试" name="timeoutRetry" :disabled="isMacroVisible || !isAllPrivilege">
                            <service-governance :index-id="indexId" :template-info="templateInfo"></service-governance>
                        </el-tab-pane>
                    </el-tabs>
                </el-row>
            </div>
        </div>
        <div v-if="isVersionContentVisible">
            <json-diff :is-overwrite="true" :json-diff-info="versionDiffInfo" @overwrite-operate="overwriteVersion" @close-dialog="closeVersionContentDialog"></json-diff>
        </div>
        <div v-if="isEditSaveVisible">
            <template-save-edit-dialog :index-id="indexId" :template-info="templateInfo" @close-edit-save-dialog="closeEditSaveDialog" @edit-save-success="editSaveSuccess"></template-save-edit-dialog>
        </div>
        <div v-if="isTemplateConfigVisible">
            <template-config-dialog
            :metadata-list="metadataList"
            @cover-content="coverConfigTemplate"
            @close-dialog="closeTemplateConfigDialog">
            </template-config-dialog>
        </div>
        <div v-if="isTemplateInsertVisible">
            <template-insert-dialog
            :field-list="fieldList"
            @insert-template-content="insertTemplateContent"
            @close-dialog="closeTemplateInsertDialog">
            </template-insert-dialog>
        </div>
    </div>
</template>

<script>
import '../../../../components';
import ProfileContent from './profile_content';
import TemplateTest from './template_test/template_test';
import ServiceGovernance from './service_governance/service_governance';
import TemplateSaveEditDialog from './template_save_edit_dialog/template_save_edit_dialog';
import TemplateConfigDialog from './template_config_dialog';
import TemplateInsertDialog from './template_insert_dialog';

export default {
  props: ['indexId', 'indexName', 'metadataList', 'fieldList', 'clusters', 'isAllPrivilege', 'templateInfo', 'macroList', 'temPanelHeight'],
  data() {
    return {
      loading: false,
      clusterId: this.clusters[0].id,
      activeTab: 'edit',
      selectedMacro: '',
      resultContent: '',
      explainContent: '',
      apiContent: {},
      paramsInfo: [],
      historyVersionList: [],
      isShowHistoryVersion: false,
      isVersionContentVisible: false,
      versionDiffInfo: {},
      datasourceList: {},
      datasourceId: '',
      sql: '',
      isEditSaveVisible: false,
      isTemplateConfigVisible: false,
      isTemplateInsertVisible: false,
      profileData: {
        profile: {
          shards: [],
        },
      },
      isProfileVisible: false,
    };
  },
  methods: {
    copySuccess() {
      this.$message.success('已复制内容到剪贴板！');
    },
    copyError(e) {
      this.$message.errorMessage(`复制appSecret失败！${e}`);
    },
    queryProfile() {
      const params = {
        indexId: this.indexId,
        templateName: this.templateInfo.templateName,
        params: this.templateInfo.params,
        clusterId: this.clusterId,
        profile: true,
      };
      this.loading = true;
      this.$http.post('/index_template/debug.json', params).then((data) => {
        if (data.substring(2, 7) === 'error') {
          this.resultContent = JSON.stringify(JSON.parse(data), undefined, 2);
        } else {
          const resultData = JSON.parse(data);
          this.profileData.profile.shards = resultData.profile.shards.map((obj) => {
            const rObj = { ...obj };
            const totalNum = obj.searches[0].query.reduce((accumulator, currentValue) =>
            accumulator + Number(currentValue.time.replace(/([0-9]+\.[0-9]*)ms/, '$1')), 0);
            rObj.totalTime = totalNum.toFixed(3);
            return rObj;
          });
          this.profileData.profile.shards.sort((a, b) => Number(b.totalTime) - Number(a.totalTime));
          this.isProfileVisible = true;
        }
      })
      .finally(() => {
        this.loading = false;
      });
    },
    insertTemplate() {
      this.isTemplateInsertVisible = true;
    },
    insertTemplateContent(content) {
      this.$refs.aceEditor1.editor.insert(content);
      this.closeTemplateInsertDialog();
    },
    closeTemplateInsertDialog() {
      this.isTemplateInsertVisible = false;
    },
    setCustomTemplate() {
      this.isTemplateConfigVisible = true;
    },
    coverConfigTemplate(data) {
      this.templateInfo.content = data;
      this.closeTemplateConfigDialog();
    },
    closeTemplateConfigDialog() {
      this.isTemplateConfigVisible = false;
    },
    initSql(dsId) {
      const tb = this.datasourceList[dsId].split('/')[2];
      this.sql = `select * from ${tb}`;
    },
    tabClick() {
      if (this.activeTab === 'api') {
        const params = {
          indexId: this.indexId,
          templateName: this.templateInfo.templateName,
        };
        this.loading = true;
        this.$http.post('/index_template/genapi.json', params).then((data) => {
          this.apiContent = data;
        })
        .finally(() => {
          this.loading = false;
        });
      } else if (this.activeTab === 'test') {
        const params = {
          indexId: this.indexId,
          templateName: this.templateInfo.templateName,
        };
        this.loading = true;
        this.$http.post('/index_template/performance_script/param.json', params).then((data) => {
          this.paramsInfo = data.map((obj) => {
            const rObj = {};
            rObj.paramName = obj.paramName;
            rObj.include = false;
            rObj.valueType = '1';
            rObj.value = '';
            return rObj;
          });
        })
        .finally(() => {
          this.loading = false;
        });
      }
    },
    insertMacro() {
      if (this.selectedMacro) {
        this.$refs.aceEditor1.editor.insert(this.insertContent);
      }
    },
    changeEditContent(val) {
      if (this.templateInfo.content !== val) {
        this.templateInfo.content = val;
      }
    },
    changeDebugContent(val) {
      if (this.templateInfo.params !== val) {
        this.templateInfo.params = val;
      }
    },
    changeSqlContent(val) {
      if (this.sql !== val) {
        this.sql = val;
      }
    },
    handleExecute() {
      const dataParams = {
        indexId: this.indexId,
        sql: this.sql,
        datasourceId: this.datasourceId,
      };
      this.loading = true;
      this.$http.post('/index_template/execute.json', dataParams).then((data) => {
        try {
          this.explainContent = JSON.stringify(data, undefined, 2);
        } catch (e) {
          this.explainContent = `解析错误: ${data.result}`;
        }
      })
      .finally(() => {
        this.loading = false;
      });
    },
    handleResetParams() {
      this.templateInfo.params =
      JSON.stringify(JSON.parse(this.templateInfo.resetParams), undefined, 2);
    },
    handleFormatParams() {
      const p = JSON.parse(this.templateInfo.params);
      if (p.params && p.params !== null) {
        this.templateInfo.params = JSON.stringify(p.params, undefined, 2);
      } else {
        this.templateInfo.params = JSON.stringify(p, undefined, 2);
      }
    },
    handleExplain() {
      const dataParams = {
        sql: this.sql,
        clusterId: this.clusterId,
      };
      this.loading = true;
      this.$http.post('/index_template/explain.json', dataParams).then((data) => {
        try {
          this.explainContent = JSON.stringify(JSON.parse(data), undefined, 2);
        } catch (e) {
          this.explainContent = `解析错误: ${data.result}`;
        }
      })
      .finally(() => {
        this.loading = false;
      });
    },
    handleDelete() {
      this.$message.confirmMessage(`确定删除模板${this.templateInfo.templateName}吗?`, () => {
        const params = {
          indexId: this.indexId,
          indexName: this.indexName,
          templateId: this.templateInfo.id,
          templateName: this.templateInfo.templateName,
        };
        this.loading = true;
        this.$http.post('/index_template/delete.json', params).then(() => {
          this.$message.successMessage('删除成功', () => {
            this.$emit('close-delete');
          });
        })
        .finally(() => {
          this.loading = false;
        });
      });
    },
    handleSave() {
      const params = {
        indexId: this.indexId,
        templateName: this.templateInfo.templateName,
        content: this.templateInfo.content,
        params: this.templateInfo.params,
      };
      this.loading = true;
      this.$http.post('/index_template/update.json', params).then(() => {
        this.$message.successMessage('保存成功', () => {
          this.$emit('close-edit');
        });
      })
      .finally(() => {
        this.loading = false;
      });
    },
    handleApprove() {
      this.isEditSaveVisible = true;
    },
    closeEditSaveDialog() {
      this.isEditSaveVisible = false;
    },
    editSaveSuccess() {
      this.isEditSaveVisible = false;
      this.$emit('close-edit');
      this.getHistoryList();
    },
    handleRender() {
      const dataParams = {
        indexId: this.indexId,
        templateName: this.templateInfo.templateName,
        params: this.templateInfo.params,
        clusterId: this.clusterId,
      };
      this.loading = true;
      this.$http.post('/index_template/render.json', dataParams).then((data) => {
        try {
          this.resultContent = this.$common.JSONbigStringifyFormat(this.$common.JSONbigParse(data));
        } catch (e) {
          this.resultContent = `解析错误: ${data}`;
        }
      })
      .finally(() => {
        this.isProfileVisible = false;
        this.loading = false;
      });
    },
    handleDebug() {
      const dataParams = {
        indexId: this.indexId,
        templateName: this.templateInfo.templateName,
        params: this.templateInfo.params,
        clusterId: this.clusterId,
      };
      this.loading = true;
      this.$http.post('/index_template/debug.json', dataParams).then((data) => {
        this.resultContent = this.$common.JSONbigStringifyFormat(this.$common.JSONbigParse(data));
      })
      .finally(() => {
        this.isProfileVisible = false;
        this.loading = false;
      });
    },
    handleHistoryVersion() {
      this.isShowHistoryVersion = !this.isShowHistoryVersion;
      if (this.isShowHistoryVersion) {
        this.getHistoryList();
      }
    },
    handleVersionDialog(row) {
      this.isVersionContentVisible = true;
      this.versionDiffInfo.left = row.content;
      this.versionDiffInfo.right = this.templateInfo.content;
    },
    closeVersionContentDialog() {
      this.isVersionContentVisible = false;
    },
    overwriteVersion() {
      this.templateInfo.content = this.versionDiffInfo.left;
      this.isVersionContentVisible = false;
    },
    getHistoryList() {
      this.loading = true;
      this.$http.get(`/index_template/hislist.json?templateId=${this.templateInfo.id}`).then((data) => {
        this.historyVersionList = data;
      })
      .finally(() => {
        this.loading = false;
      });
    },
    getDataSourceList() {
      return this.$http.post('/index/loadDbList.json', { indexId: this.indexId }).then((data) => {
        if (data) {
          this.datasourceList = data;
          if (!this.datasourceList && this.datasourceList.length !== 0) {
            this.datasourceId = Object.entries(data)[0][0];
          }
        }
      });
    },
    init() {
      this.loading = true;
      Promise.all([this.getDataSourceList()]).then()
      .finally(() => {
        this.loading = false;
      });
    },
  },
  components: {
    'template-test': TemplateTest,
    'profile-content': ProfileContent,
    'template-save-edit-dialog': TemplateSaveEditDialog,
    'service-governance': ServiceGovernance,
    'template-config-dialog': TemplateConfigDialog,
    'template-insert-dialog': TemplateInsertDialog,
  },
  created() {
    this.init();
  },
  computed: {
    insertContent() {
      return `##__${this.selectedMacro}__##`;
    },
    eidtorId() {
      return `${this.templateInfo.templateName}_edit`;
    },
    debugId() {
      return `${this.templateInfo.templateName}_debug`;
    },
    isMacroVisible() {
      return this.templateInfo.type === 0;
    },
    historyVersionBtn() {
      return this.isShowHistoryVersion ? '隐藏历史版本' : '历史版本';
    },
    isEditOperate() {
      return this.activeTab === 'edit';
    },
    templateType() {
      if (this.templateInfo.type === 0) {
        return '宏';
      }
      return '模板';
    },
  },
};

</script>

<style type="text/css">
.template-content .template-title {
  color: #fff;
  line-height: 30px;
}
.template-content .template-title .template-name{
  color: #32cd32;
  font-weight: bold;
  margin-right: 15px;
}
.template-content .template-title .template-approving {
  color: red;
  margin-left: 10px;
  font-weight: initial;
}
.template-content .template-title .template-approving a {
  color: red;
}
.template-content .template-title .template-approving a:hover {
  color: rgba(255, 0, 0, 0.7);
}
.result-content .el-textarea__inner {
    background-color: #272822;
}
.api-content {
  padding: 5px;
  background-color: #222;
  color: #fff
}
.no-border {
  border: none;
}
.debug-title {
  width: 100%;
  margin-bottom: 5px;
  height: 30px;
  line-height: 30px;
}
.template-edit-content {
  height: 650px;
  width: 100%;
  overflow-y: auto; 
  float:left;
}
.template-edit-and-version-content {
  height: 650px;
  width: 62%;
  float:left;
}
.template-history-version-content {
  position: relative;
  width: 38%;
  float:left;
}
.template-history-version-content table {
  font-size: 10px;
}
.template-history-version-content .el-table .cell {
  line-height: initial;
}
.template-history-version-content .el-table tr:hover {
  cursor: pointer;
}
.render-cluster {
  margin-left: 15px;
}
.render-cluster >span {
  font-size: 16px;
}
</style>

<template>
  <div>
    <Card shadow>
      <Form ref="searchForm"
            :model="pageInfo"
            inline
            :label-width="80">
        <FormItem label="请求路径" prop="path">
          <Input type="text" v-model="pageInfo.path" placeholder="请输入关键字"/>
        </FormItem>
        <FormItem label="接口名称" prop="apiName">
          <Input type="text" v-model="pageInfo.apiName" placeholder="请输入关键字"/>
        </FormItem>
        <FormItem label="接口编码" prop="apiCode">
          <Input type="text" v-model="pageInfo.apiCode" placeholder="请输入关键字"/>
        </FormItem>
        <FormItem label="服务名" prop="serviceId">
          <Input type="text" v-model="pageInfo.serviceId" placeholder="请输入关键字"/>
        </FormItem>
        <FormItem>
          <Button type="primary" @click="handleSearch(1)">查询</Button>&nbsp;
          <Button @click="handleResetForm('searchForm')">重置</Button>
        </FormItem>
      </Form>
      <div class="search-con search-con-top">
        <ButtonGroup>
          <Button :disabled="hasAuthority('systemApiEdit')?false:true"  class="search-btn" type="primary" @click="handleModal()">
            <span>添加</span>
          </Button>
        </ButtonGroup>
      </div>
      <Alert show-icon>
        <span>自动扫描<code>@EnableResourceServer</code>资源服务器接口信息,注:自动添加的接口,都是未公开的. <code>只有公开的接口,才可以通过网关访问。否则将提示:"请求地址,拒绝访问!"</code></span>
      </Alert>
      <Table border :columns="columns" :data="data" :loading="loading">
        <template slot="apiName" slot-scope="{ row }">
          <span>{{row.apiName}}</span>
        </template>
        <template slot="isAuth" slot-scope="{ row }">
          <Tag color="blue" v-if="row.isOpen===1">公开的</Tag>
          <Tag v-else-if="row.isOpen!==1">内部的</Tag>
          <Tag color="green" v-if="row.isAuth===1">身份认证</Tag>
          <Tag v-else-if="row.isAuth!==1">无需认证</Tag>
          <Tag v-if="row.status===1"   color="green">启用</Tag>
          <Tag v-else-if="row.status===2"   color="orange">维护中</Tag>
          <Tag v-else=""  color="red">禁用</Tag>
        </template>
        <template slot="action" slot-scope="{ row }">
          <a :disabled="hasAuthority('systemApiEdit')?false:true"   @click="handleModal(row)">
            编辑</a>&nbsp;
          <a :disabled="hasAuthority('systemApiEdit')?false:true"   @click="handleRemove(row)">
            删除</a>
        </template>
      </Table>
      <Page transfer :total="pageInfo.total" :current="pageInfo.page" :page-size="pageInfo.limit" show-elevator
            show-sizer
            show-total
            @on-change="handlePage" @on-page-size-change='handlePageSize'></Page>
    </Card>
    <Modal v-model="modalVisible"
           :title="modalTitle"
           width="40"
           @on-cancel="handleReset">
      <div>
        <Alert show-icon v-if="formItem.apiId?true:false">
          <span>自动扫描接口swagger注解。</span>
          <Poptip placement="bottom" title="示例代码">
            <a>示例代码</a>
            <div slot="content">
              <div v-highlight>
                <pre>
                      // 接口介绍
                      @ApiOperation(value = "接口名称", notes = "接口备注")
                      @PostMapping("/testApi")
                      // 忽略接口,将不再添加或修改次接口
                      @ApiIgnore
                      public ResultBody testApi() {
                          return ResultBody.success();
                      }
                </pre>
              </div>
            </div>
          </Poptip>
        </Alert>
        <Form ref="form1" :model="formItem" :rules="formItemRules" :label-width="100">
          <FormItem label="服务名称" prop="serviceId">
            <Select :disabled="formItem.apiId && formItem.isPersist === 1?true:false" v-model="formItem.serviceId" filterable clearable>
              <Option v-for="item in selectServiceList" :value="item.serviceId">{{ item.serviceName }}</Option>
            </Select>
          </FormItem>
          <FormItem label="接口分类" prop="apiCategory">
            <Input v-model="formItem.apiCategory" placeholder="请输入内容"></Input>
          </FormItem>
          <FormItem label="接口标识" prop="apiCode">
            <Input :disabled="formItem.apiId && formItem.isPersist === 1?true:false" v-model="formItem.apiCode" placeholder="请输入内容"></Input>
          </FormItem>
          <FormItem label="接口名称" prop="apiName">
            <Input :disabled="formItem.apiId && formItem.isPersist === 1?true:false" v-model="formItem.apiName" placeholder="请输入内容"></Input>
          </FormItem>
          <FormItem label="请求地址" prop="path">
            <Input :disabled="formItem.apiId && formItem.isPersist === 1?true:false" v-model="formItem.path" placeholder="请输入内容"></Input>
          </FormItem>
          <FormItem label="优先级">
            <InputNumber v-model="formItem.priority"></InputNumber>
          </FormItem>
          <FormItem label="身份认证">
            <RadioGroup  v-model="formItem.isAuth">
              <Radio :disabled="formItem.apiId && formItem.isPersist === 1?true:false" label="0">否</Radio>
              <Radio :disabled="formItem.apiId && formItem.isPersist === 1?true:false" label="1">是</Radio>
            </RadioGroup>
          </FormItem>
          <FormItem label="公开访问">
            <RadioGroup  v-model="formItem.isOpen">
              <Radio label="0">否</Radio>
              <Radio label="1">是</Radio>
            </RadioGroup>
            <Tooltip content="接口是否可以通过网关访问">
              <Icon type="ios-alert" size="16"/>
            </Tooltip>
          </FormItem>
          <FormItem label="状态">
            <RadioGroup v-model="formItem.status">
              <Radio label="0">禁用</Radio>
              <Radio label="1">启用</Radio>
              <Radio label="2">维护中</Radio>
            </RadioGroup>
          </FormItem>
          <FormItem label="描述">
            <Input v-model="formItem.apiDesc" type="textarea" placeholder="请输入内容"></Input>
          </FormItem>
        </Form>
        <div class="drawer-footer">
          <Button type="default" @click="handleReset">取消</Button>&nbsp;
          <Button type="primary" @click="handleSubmit" :loading="saving">保存</Button>
        </div>
      </div>
    </Modal>

  </div>
</template>

<script>
  import {listConvertTree} from '@/libs/util'
  import {getApis, updateApi, addApi, removeApi} from '@/api/api'
  import {getServiceList} from '@/api/gateway'

  export default {
    name: 'SystemApi',
    data () {
      const validateEn = (rule, value, callback) => {
        let reg = /^[_.a-zA-Z0-9]+$/
        if (value === '') {
          callback(new Error('接口标识不能为空'))
        } else if (value !== '' && !reg.test(value)) {
          callback(new Error('只允许字母、数字、点、下划线'))
        } else {
          callback()
        }
      }
      return {
        loading: false,
        modalVisible: false,
        modalTitle: '',
        saving: false,
        pageInfo: {
          total: 0,
          page: 1,
          limit: 10,
          path: '',
          apiName: '',
          apiCode: '',
          serviceId: ''
        },
        selectServiceList: [],
        formItemRules: {
          serviceId: [
            {required: true, message: '所属服务不能为空', trigger: 'blur'}
          ],
          apiCategory: [
            {required: true, message: '接口分类不能为空', trigger: 'blur'}
          ],
          path: [
            {required: true, message: '接口地址不能为空', trigger: 'blur'}
          ],
          apiCode: [
            {required: true, validator: validateEn, trigger: 'blur'}
          ],
          apiName: [
            {required: true, message: '接口名称不能为空', trigger: 'blur'}
          ]
        },
        formItem: {
          apiId: '',
          apiCode: '',
          apiName: '',
          apiCategory: 'default',
          path: '',
          status: 1,
          isAuth: 1,
          openSwatch: false,
          authSwatch: true,
          serviceId: '',
          priority: 0,
          apiDesc: '',
          isOpen:1
        },
        columns: [
          {
            type: 'selection',
            width: 60,
            align: 'center'
          },
          {
            title: '名称',
            key: 'apiName',
            slot: 'apiName',
            width: 200,
            filters: [
              {
                label: '禁用',
                value: 0
              },
              {
                label: '启用',
                value: 1
              }
            ],
            filterMultiple: false,
            filterMethod (value, row) {
              if (value === 0) {
                return row.status === 0
              } else if (value === 1) {
                return row.status === 1
              }
            }
          },
          {
            title: '地址',
            key: 'path',
            width: 200,
          },
          {
            title: '分类',
            key: 'apiCategory',
            width: 100,
          },
          {
            title: '服务名称',
            key: 'serviceId',
            width: 200
          },
          {
            title: '接口安全',
            key: 'isAuth',
            slot: 'isAuth',
            width: 250
          },
          {
            title: '描述',
            key: 'apiDesc',
            width: 400
          },
          {
            title: '最后更新时间',
            key: 'updateTime',
            width: 180
          },
          {
            title: '操作',
            key: '',
            slot: 'action',
            fixed: 'right',
            width: 120
          }
        ],
        data: []
      }
    },
    methods: {
      handleModal (data) {
        if (data) {
          this.modalTitle = '编辑接口 - ' + data.apiName
          this.formItem = Object.assign({}, this.formItem, data)
        } else {
          this.modalTitle = '添加接口'
        }
        this.formItem.status = this.formItem.status + ''
        this.formItem.isAuth = this.formItem.isAuth + ''
        this.formItem.isOpen = this.formItem.isOpen + ''
        this.modalVisible = true
      },
      handleResetForm (form) {
        this.$refs[form].resetFields()
      },
      handleReset () {
        const newData = {
          apiId: '',
          apiCode: '',
          apiName: '',
          apiCategory: 'default',
          path: '',
          status: 1,
          isAuth: 1,
          serviceId: '',
          priority: 0,
          apiDesc: '',
          isOpen:1
        }
        this.formItem = newData
        //重置验证
        this.handleResetForm('form1')
        this.modalVisible = false
        this.saving = false
      },
      handleSubmit () {
        this.$refs['form1'].validate((valid) => {
          if (valid) {
            this.saving = true
            if (this.formItem.apiId) {
              updateApi(this.formItem).then(res => {
                if (res.code === 0) {
                  this.$Message.success('保存成功')
                }
                this.handleReset()
                this.handleSearch()
              }).finally(() => {
                this.saving = false
              })
            } else {
              addApi(this.formItem).then(res => {
                if (res.code === 0) {
                  this.$Message.success('保存成功')
                }
                this.handleReset()
                this.handleSearch()
              }).finally(() => {
                this.saving = false
              })
            }
          }
        })
      },
      handleRemove (data) {
        this.$Modal.confirm({
          title: '确定删除吗？',
          onOk: () => {
            removeApi(data.apiId).then(res => {
              this.handleSearch()
              if (res.code === 0) {
                this.pageInfo.page = 1
                this.$Message.success('删除成功')
              }
            })
          }
        })
      },
      handleSearch (page) {
        if (page) {
          this.pageInfo.page = page
        }
        this.loading = true
        getApis(this.pageInfo).then(res => {
          this.data = res.data.records
          this.pageInfo.total = parseInt(res.data.total)
        }).finally(() => {
          this.loading = false
        })
      },
      handlePage (current) {
        this.pageInfo.page = current
        this.handleSearch()
      },
      handlePageSize (size) {
        this.pageInfo.limit = size
        this.handleSearch()
      },
      handleLoadServiceList () {
        getServiceList().then(res => {
          if (res.code === 0) {
            this.selectServiceList = res.data
          }
        })
      }
    },
    mounted: function () {
      this.handleLoadServiceList()
      this.handleSearch()
    }
  }
</script>

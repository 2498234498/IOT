<template>
  <el-form ref="form" :model="form" :rules="rules">
    <div class="name-container">
      <el-form-item label="名称" prop="name">
        <el-input multiple v-model="form.name"></el-input>
      </el-form-item>
      <el-form-item prop="debugMode">
        <el-checkbox v-model="form.debugMode">调试模式</el-checkbox>
      </el-form-item>
    </div>
    <el-form-item label="客户名称模式" prop="customerNamePattern">
      <el-input v-model="form.customerNamePattern"></el-input>
      <span class="desc">客户名称模式,使用 ${metaKeyName} 替换元数据中的变量</span>
    </el-form-item>
    <el-form-item label="客户缓存过期时间(秒)" prop="customerCacheExpiration">
      <el-input type="number" :min="0" v-model="form.customerCacheExpiration"></el-input>
      <span class="desc">指定允许存储找到的客户记录的最大时间间隔0值表示记录将永不过期</span>
    </el-form-item>
    <el-form-item label="描述" prop="description">
      <el-input type="textarea" v-model="form.description"></el-input>
    </el-form-item>
  </el-form>
</template>

<script>
export default {
  props: {
    nodeInfo: {
      type: Object
    }
  },
  data () {
    const customerCacheExpiration = (rule, value, callback) => {
      if (Number(value) < 0) {
        callback(new Error('客户缓存过期时间应大于或等于0'))
      } else if (value === '' || value === undefined) {
        callback(new Error('客户缓存过期时间不能为空'))
      } else {
        callback()
      }
    }
    return {
      form: {
        name: '',
        debugMode: '',
        description: '',
        customerNamePattern: '',
        customerCacheExpiration: ''
      },
      rules: {
        name: [{ required: true, message: '名称不能为空', trigger: 'change' }],
        customerNamePattern: [{ required: true, message: '客户名称模式不能为空', trigger: 'change' }],
        customerCacheExpiration: [{ required: true, validator: customerCacheExpiration, trigger: 'change' }]
      }
    }
  },
  methods: {
    submit () {
      this.$refs.form.validate(valid => {
        if (!valid) return false
        this.$emit('submit', {
          debugMode: this.form.debugMode || false,
          name: this.form.name,
          configuration: {
            customerNamePattern: this.form.customerNamePattern,
            customerCacheExpiration: this.form.customerCacheExpiration
          },
          additionalInfo: {
            description: this.form.description
          },
          tplType: Object.is(JSON.stringify(this.nodeInfo), '{}') || 'edit'
        })
      })
    },
    init () {
      const { name, debugMode } = this.nodeInfo
      const { customerNamePattern, customerCacheExpiration } = this.nodeInfo.configuration || {}
      const { description } = this.nodeInfo.additionalInfo || {}
      this.form = {
        name: name || '',
        debugMode: debugMode || false,
        description: description || '',
        customerNamePattern,
        customerCacheExpiration: JSON.stringify(this.nodeInfo) === '{}' ? 300 : customerCacheExpiration
      }
      console.log(this.form)
    }
  },
  created () {
    this.init()
  }
}
</script>

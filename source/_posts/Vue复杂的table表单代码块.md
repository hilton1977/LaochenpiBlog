---
title: Vue复杂的table表单代码块
date: 2019-07-17 10:51:26
tags:
	- 前端
	- 技术
    
![](/images/vue.jpg)
> 记录项目中遇到的Element UI 中遇到的各种情况 代码块

### Vue 代码块
``` vue
<el-form :model="refundCashDetail" ref="refundCashDetail">
          <el-table
            :data="refundCashDetail.refundPsgList"
            size="mini"
            border
            style="width: 100%">
            <el-table-column width="60" label="序号" align="center">
              <template slot-scope="scope">
                {{ scope.$index + 1 }}
              </template>
            </el-table-column>
            <el-table-column prop="name" label="姓名" align="center"></el-table-column>
            <el-table-column prop="idNumber" label="证件号码" align="center"></el-table-column>
            <el-table-column prop="applyCash" label="申请退定金" align="center"
                             v-if="refundCashDetail.orderDetail.refundType===1"></el-table-column>
            <el-table-column label="实际退定金" align="center" v-if="refundCashDetail.orderDetail.refundType===1">
              <template slot-scope="scope">
                <el-form-item
                  :prop="'refundPsgList.'+ scope.$index +'.practicalCash'"
                  :inline-message="true"
                  :rules="[{ type: 'number',required:true,message:'金额不能为空',trigger: 'blur'}]">
                  <el-input-number
                                   :placeholder="scope.row.oneDeposit+'/每人'"
                                   v-model.number="scope.row.practicalCash"
                                   :max="refundCashDetail.orderDetail.oneDeposit"
                                   autocomplete="off"
                                   :min="0"/>
                </el-form-item>
              </template>
            </el-table-column>
          </el-table>
        </el-form>
```
这里 `form` 和 `table`配合使用其中`refundPsgList`循环出来的元素在数据双向绑定上`prop`的值需要特殊改造，使用`scope.$index`来进行绑定，一般情况下`prop`和对应的`v-model`保持一致
---

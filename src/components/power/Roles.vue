<template>
  <div>
    <!-- 面包屑导航 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片视图 -->
    <el-card>
      <!-- 角色添加按钮区 -->
      <el-row>
        <el-col>
          <el-button type="primary" @click="dialogVisible = true">添加角色</el-button>
        </el-col>
      </el-row>
      <!-- 角色列表区 -->
      <el-table :data="rolesList" border stripe>
        <!-- 展开列 -->
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-row
              :class="['bd_bottom', i1 == 0 ? 'bd_top':'', 'vcenter']"
              v-for="(item1, i1) in scope.row.children"
              :key="item1.id"
            >
              <!-- 渲染一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="removeRightById(scope.row, item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 渲染二级和三级权限 -->
              <el-col :span="19">
                <el-row
                  :class="[i2 == 0 ? '':'bd_top', 'vcenter']"
                  v-for="(item2, i2) in item1.children"
                  :key="item2.id"
                >
                  <el-col :span="6">
                    <el-tag
                      type="success"
                      closable
                      @close="removeRightById(scope.row, item2.id)"
                    >{{item2.authName}}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="18">
                    <el-tag
                      type="warning"
                      v-for="(item3, i3) in item2.children"
                      :key="item3.id"
                      closable
                      @close="removeRightById(scope.row, item3.id)"
                    >{{item3.authName}}</el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <!-- 索引列 -->
        <el-table-column type="index"></el-table-column>
        <el-table-column label="角色名称" prop="roleName"></el-table-column>
        <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
        <el-table-column label="操作" width="300">
          <template slot-scope="scope">
            <el-button type="primary" icon="el-icon-edit" size="mini" @click="getRole(scope.row)">编辑</el-button>
            <el-button
              type="danger"
              icon="el-icon-delete"
              size="mini"
              @click="removeRoleById(scope.row.id)"
            >删除</el-button>
            <el-button
              type="warning"
              icon="el-icon-search"
              size="mini"
              @click="showSetRightDialog(scope.row)"
            >分配权限</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
    <!-- 添加角色区域 -->
    <el-dialog title="添加角色" :visible.sync="dialogVisible" width="40%" @close="addDialogClosed">
      <el-form :model="addRole" :rules="addFormRules" ref="addRoleFormRef" label-width="100px">
        <el-form-item label="角色姓名" prop="roleName">
          <el-input v-model="addRole.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model="addRole.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addNewRole">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 编辑用户区域 -->
    <el-dialog title="编辑角色" :visible.sync="editDialog" width="40%">
      <el-form :model="role" :rules="addFormRules" ref="editFormRef" label-width="100px">
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="role.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc">
          <el-input v-model="role.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="editDialog = false">取 消</el-button>
        <el-button type="primary" @click="editRolesInfo">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 分配权限区域 -->
    <el-dialog title="权限分配" :visible.sync="rightDialog" width="40%" @close="rightDialogClosed">
      <el-tree
        :data="rightsList"
        :props="defaultProps"
        show-checkbox
        :default-expand-all="true"
        :default-checked-keys="defKeys"
        node-key="id"
        ref="treeRef"
      ></el-tree>
      <span slot="footer">
        <el-button @click="rightDialog = false">取 消</el-button>
        <el-button type="primary" @click="allotRights">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      rolesList: [],
      rightsList: [],
      defaultProps: {
        label: 'authName',
        children: 'children'
      },
      defKeys: [],
      roleId: '',
      role: {},
      addRole: {
        roleName: '',
        roleDesc: ''
      },
      addFormRules: {
        roleName: [
          {
            required: true,
            message: '请输入角色名称！',
            trigger: 'blur'
          }
        ],
        roleDesc: [
          {
            required: true,
            message: '请输入角色描述！',
            trigger: 'blur'
          }
        ]
      },
      editDialog: false,
      dialogVisible: false,
      rightDialog: false
    }
  },
  created() {
    this.getRolesList()
  },
  methods: {
    async getRolesList() {
      const { data: res } = await this.$http.get('roles')
      if (res.meta.status !== 200) {
        return this.$msg.error('获取角色列表失败！')
      }
      //this.$msg.success('获取角色列表成功！')
      this.rolesList = res.data
    },
    async getRole(row) {
      const { data: res } = await this.$http.get('roles/' + row.id)
      if (res.meta.status !== 200) {
        return this.$msg.error('获取角色失败！')
      }
      this.role = res.data
      this.editDialog = true
    },
    editRolesInfo() {
      this.$refs.editFormRef.validate(async valid => {
        if (!valid) {
          return
        }
        const { data: res } = await this.$http.put(
          `roles/${this.role.roleId}`,
          {
            roleName: this.role.roleName,
            roleDesc: this.role.roleDesc
          }
        )
        if (res.meta.status !== 200) {
          return this.$msg.error('修改角色失败')
        }
        this.getRolesList()
        this.editDialog = false
      })
    },
    addDialogClosed() {
      this.$refs.addRoleFormRef.resetFields()
    },
    addNewRole() {
      this.$refs.addRoleFormRef.validate(async valid => {
        if (!valid) {
          return
        }
        const { data: res } = await this.$http.post('roles', {
          roleName: this.addRole.roleName,
          roleDesc: this.addRole.roleDesc
        })
        if (res.meta.status !== 201) {
          return this.$msg.error('修改失败！')
        }
        this.$msg.success('修改成功！')
        this.getRolesList()
        this.dialogVisible = false
      })
    },
    async removeRoleById(id) {
      const result = await this.$confirm(
        '此操作将永久删除该角色, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch(err => err)
      if (result !== 'confirm') {
        return this.$msg.info('已取消删除')
      }
      const { data: res } = await this.$http.delete('roles/' + id)
      if (res.meta.status !== 200) {
        return this.$msg.error('删除用户失败！')
      }
      this.$msg.success('删除用户成功！')
      this.getRolesList()
    },
    async removeRightById(role, id) {
      const result = await this.$confirm(
        '此操作将永久删除该权限, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch(err => err)
      if (result !== 'confirm') {
        return this.$msg.info('已取消删除')
      }
      const { data: res } = await this.$http.delete(
        `roles/${role.id}/rights/${id}`
      )
      if (res.meta.status !== 200) {
        return this.$msg.error('删除用户失败！')
      }
      this.$msg.success('删除权限成功！')
      role.children = res.data
    },
    async showSetRightDialog(role) {
      this.roleId = role.id
      const { data: res } = await this.$http.get('rights/tree')
      if (res.meta.status !== 200) {
        return this.$msg.error('获取权限树失败')
      }
      this.rightsList = res.data
      this.getLeafKeys(role, this.defKeys)
      this.rightDialog = true
    },
    rightDialogClosed() {
      this.defKeys = []
      this.rightDialog = false
    },
    getLeafKeys(node, array) {
      if(!node.children) {
        return array.push(node.id)
      }
      node.children.forEach((item) => {
        this.getLeafKeys(item, array)
      })
    },
    async allotRights() {
      const keys = [...this.$refs.treeRef.getCheckedKeys(),...this.$refs.treeRef.getHalfCheckedKeys()]
      const idStr = keys.join(',')
      const {data:res} = await this.$http.post(`roles/${this.roleId}/rights`, {rids: idStr})
      if(res.meta.status !== 200) {
        return this.$msg.error('分配权限失败！')
      }
      this.$msg.success('分配权限成功！')
      this.getRolesList()
      this.rightDialog = false
    }
  }
}
</script>

<style lang="less" scoped>
.el-table {
  margin-top: 20px;
}
.el-tag {
  margin: 10px;
}
.bd_top {
  border-top: 1px solid #eee;
}
.bd_bottom {
  border-bottom: 1px solid #eee;
}
.vcenter {
  display: flex;
  align-items: center;
}
</style>
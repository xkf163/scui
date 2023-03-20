<template>
	<el-form ref="loginForm" :model="form" :rules="rules" label-width="0" size="large" @keyup.enter="login">
		<el-form-item prop="user">
			<el-input v-model="form.user" prefix-icon="el-icon-user" clearable :placeholder="$t('login.userPlaceholder')">
				<template #append>
					<el-select v-model="userType" style="width: 130px;">
						<el-option :label="$t('login.admin')" value="admin"></el-option>
						<el-option :label="$t('login.user')" value="user"></el-option>
					</el-select>
				</template>
			</el-input>
		</el-form-item>
		<el-form-item prop="password">
			<el-input v-model="form.password" prefix-icon="el-icon-lock" clearable show-password :placeholder="$t('login.PWPlaceholder')"></el-input>
		</el-form-item>

		<el-form-item prop="inputCode">
			<el-col :span="16">
				<el-input v-model="form.inputCode" prefix-icon="el-icon-help"  size="large" type="text" placeholder="请输入验证码">
<!--					<a-icon slot="prefix" type="smile" :style="{ color: 'rgba(0,0,0,.25)' }"/>-->
				</el-input>
			</el-col>
			<el-col :span="8" style="text-align: right">
				<img v-if="requestCodeSuccess" :src="randCodeImage" @click="handleChangeCheckCode"/>
				<img v-else styl src="../../../assets/checkcode.png" @click="handleChangeCheckCode"/>
			</el-col>
		</el-form-item>



		<el-form-item style="margin-bottom: 10px;">
				<el-col :span="12">
					<el-checkbox :label="$t('login.rememberMe')" v-model="form.autologin"></el-checkbox>
				</el-col>
				<el-col :span="12" class="login-forgot">
					<router-link to="/reset_password">{{ $t('login.forgetPassword') }}？</router-link>
				</el-col>
		</el-form-item>
		<el-form-item>
			<el-button type="primary" style="width: 100%;" :loading="islogin" round @click="login">{{ $t('login.signIn') }}</el-button>
		</el-form-item>
		<div class="login-reg">
			{{$t('login.noAccount')}} <router-link to="/user_register">{{$t('login.createAccount')}}</router-link>
		</div>
	</el-form>
</template>

<script>
	import http from "@/utils/request";

	export default {
		data() {
			return {
				requestCodeSuccess: false,
				randCodeImage: '',
				currdatetime: '',
				userType: 'admin',
				form: {
					user: "admin",
					password: "123456",
					autologin: false
				},
				rules: {
					user: [
						{required: true, message: this.$t('login.userError'), trigger: 'blur'}
					],
					password: [
						{required: true, message: this.$t('login.PWError'), trigger: 'blur'}
					]
				},
				islogin: false,
			}
		},
		created() {
			this.handleChangeCheckCode();
		},
		watch:{
			userType(val){
				if(val == 'admin'){
					this.form.user = 'admin'
					this.form.password = 'admin'
				}else if(val == 'user'){
					this.form.user = 'user'
					this.form.password = 'user'
				}
			}
		},
		mounted() {

		},
		methods: {


			async login(){

				var validate = await this.$refs.loginForm.validate().catch(()=>{})
				if(!validate){ return false }

				this.islogin = true
				var data = {
					username: this.form.user,
					//password: this.$TOOL.crypto.MD5(this.form.password),
					password: this.form.password,
					captcha: this.form.inputCode,
					checkKey: this.currdatetime,
					remember_me: this.form.autologin,
				}
				//获取token
				var user = await this.$API.auth.token.post(data)
				console.log("登录成功：",user)
				if(user.code == 200){
					this.$TOOL.cookie.set("TOKEN", user.result.token, {
						expires: this.form.autologin? 24*60*60 : 0
					})
					this.$TOOL.data.set("USER_INFO", user.result.userInfo)
				}else{
					this.islogin = false
					this.$message.warning(user.message)
					return false
				}
				//获取菜单
				var menu = null
				if(this.form.user == 'admin'){
					menu = await this.$API.system.menu.myMenus.get()
				}else{
					menu = await this.$API.demo.menu.get()
				}
				if(menu.code == 0){
					if(menu.result.menu.length==0){
						this.islogin = false
						this.$alert("当前用户无任何菜单权限，请联系系统管理员", "无权限访问", {
							type: 'error',
							center: true
						})
						return false
					}
					console.log("menu",menu.result.menu)

					menu ={"code":200,"result":{"menu":[
						{"name":"home","path":"/home","meta":{"title":"首页","icon":"el-icon-eleme-filled","type":"menu"},"children":[
							{"name":"dashboard","path":"/dashboard","meta":{"title":"控制台","icon":"el-icon-menu","affix":true},"component":"home"},
							{"name":"userCenter","path":"/usercenter","meta":{"title":"帐号信息","icon":"el-icon-user","tag":"NEW"},"component":"userCenter"}]
						}]
					}}

					this.$TOOL.data.set("MENU", menu.result.menu)
					//this.$TOOL.data.set("PERMISSIONS", menu.result.permissions)
					//this.$TOOL.data.set("DASHBOARDGRID", menu.result.dashboardGrid)
				}else{
					this.islogin = false
					this.$message.warning(menu.message)
					return false
				}

				this.$router.replace({
					path: '/'
				})
				this.$message.success("Login Success 登录成功")
				this.islogin = false
			},
			/**刷新验证码*/
			handleChangeCheckCode(){
				this.currdatetime = new Date().getTime();
				this.form.inputCode = ''
				http.get(`/api/sys/randomImage/${this.currdatetime}`).then(res=>{
					if(res.success){
						this.randCodeImage = res.result
						this.requestCodeSuccess=true
					}else{
						this.$message.error(res.message)
						this.requestCodeSuccess=false
					}
				}).catch(()=>{
					this.requestCodeSuccess=false
				})
			},
		}
	}
</script>

<style>
</style>

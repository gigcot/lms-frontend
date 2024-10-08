<template>
    <v-container class="fill-height" fluid>
        <v-row align="center" justify="center">
            <v-col cols="12" sm="8" md="6">
                <v-card elevation="0" class="pa-6" :color="colors.cardBackground">
                    <v-card-title class="text-h5 mb-2">계정</v-card-title>
                    <v-divider class="mb-9"></v-divider>
                    <v-card-text align="center">
                        <p class="mb-4">로그인하거나 계정을 생성하려면 이메일 주소를 입력해 주세요</p>
                        <v-form ref="form">
                            <v-text-field v-if="isShowEmail" @keydown.enter.prevent="checkEmail" max-width="300"
                                v-model="email" label="이메일" outlined required
                                :rules="[v => !!v || '필수 항목', v => emailRegex.test(v) || '유효한 이메일 주소를 입력해 주세요']"
                                aria-label="이메일 입력"></v-text-field>
                            <span v-if="!emailRulesCheck" class="security text-medium-emphasis text-caption">
                                죄송합니다. 표기가 잘못되었습니다. name@domain.com과 같은 형식의 유효한 이메일을 입력해 주세요.
                            </span>
                            <v-text-field v-if="!isShowEmail" @keydown.enter.prevent="loginUser" type="password"
                                autocomplete="current-password" max-width="300" v-model="password" label="비밀번호" outlined
                                required :rules="passwordRules" aria-label="비밀번호 입력" ref="passwordInput"></v-text-field>
                            <span v-if="!passwordCheck" class="security text-medium-emphasis text-caption">
                                비밀번호가 잘못 입력되었습니다.
                            </span>
                            <p v-if="!isShowEmail" class="forgotPassword text-medium-emphasis text-caption"
                                @click="toggleShopPopup('forgotPassword')">비밀번호 재설정</p>
                            <div class="d-flex justify-center">
                                <v-btn v-if="isShowEmail" class="submit-button mt-4" max-width="150"
                                    @click="checkEmail">
                                    계속
                                </v-btn>
                                <v-btn v-if="!isShowEmail" class="submit-button mt-4" max-width="150"
                                    @click="loginUser">
                                    로그인
                                </v-btn>
                            </div>
                            <v-divider width="400" class="mt-9 mb-9"></v-divider>
                            <v-img class="google-login-btn" block @click="goToGoogleLogin" aria-label="구글 로그인"></v-img>
                        </v-form>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>
        <ForgotPasswordPopup v-if="currentShop === 'forgotPassword'" :email="email" @close="closeShopPopup" />
    </v-container>
</template>

<script>
import { mapActions } from 'vuex'
import router from "@/router";

import ForgotPasswordPopup from "@/popup/pages/ForgotPassword.vue";
const authenticationModule = 'authenticationModule'
const accountModule = 'accountModule'

export default {
    components: {
        ForgotPasswordPopup
    },
    data() {
        return {
            email: '',
            name: '',
            password: '',
            isShowEmail: true,
            clientId: '',
            emailRulesCheck: true,
            emailRegex: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/,
            passwordCheck: true,
            passwordRules: [
                v => !!v || '비밀번호는 필수 항목입니다.',
                v => v.length >= 8 || '비밀번호는 최소 8자 이상이어야 합니다.',
                v => /[A-Z]/.test(v) || '대문자를 포함해야 합니다.',
                v => /[a-z]/.test(v) || '소문자를 포함해야 합니다.',
                v => /[0-9]/.test(v) || '숫자를 포함해야 합니다.',
                v => /[!@#$%^&*]/.test(v) || '특수문자를 포함해야 합니다.',
            ],
            currentShop: null,
            colors: {
                cardBackground: '#fffcf7',
                buttonBackground: '#616161',
                buttonHoverBackground: '#424242',
                errorText: '#9d2a1e',
            },
        }
    },
    created() {
        this.clientId = process.env.VUE_APP_GOOGLE_CLIENT_ID;
    },
    methods: {
        ...mapActions(accountModule, ['requestEmailDuplicationCheckToDjango', 'requestNormalLoginToDjango', 'requestGoogleLoginToDjango', 'requestCreateNewSocialAccountToDjango', 'requestEmailLoginTypeToDjango']),
        ...mapActions(authenticationModule, ['requestAddRedisAccessTokenToDjango']),

        checkEmail() {
            if (this.$refs.form.validate() && this.emailRegex.test(this.email)) {
                this.checkEmailDuplication();
            } else {
                this.emailRulesCheck = false;
            }
        },
        async checkEmailDuplication() {
            try {
                const isDuplicate = await this.requestEmailDuplicationCheckToDjango(this.email.trim())
                if (isDuplicate) {
                    const response = await this.requestEmailLoginTypeToDjango(this.email.trim())
                    const LoginType = response.isLoginType
                    if (LoginType === "NORMAL") {
                        this.isShowEmail = false;
                        this.$nextTick(() => {
                            if (this.$refs.passwordInput) {
                                this.$refs.passwordInput.focus();
                            }
                        });
                    } else if (LoginType === "GOOGLE") {
                        alert('구글 회원입니다.');
                    } else {
                        alert('관리자');
                    }
                } else {
                    router.push({ path: "/account/register", query: { email: this.email.trim() } });
                }
            } catch (error) {
                alert('이메일 중복 확인 실패')
                this.isEmailValid = false
            }
        },
        async loginUser() {
            if (this.$refs.form.validate()) {
                const accountInfo = {
                    email: this.email,
                    password: this.password,
                }
                try {
                    const response = await this.requestNormalLoginToDjango(accountInfo);
                    if (response.access_token != null) {
                        const responseRedis = await this.requestAddRedisAccessTokenToDjango(this.email.trim())
                        if (responseRedis) {
                            this.$store.state.accountModule.isLoggedIn = true;
                            alert('로그인 성공!')
                            router.push('/')
                        } else {
                            alert('로그인 처리 중 오류가 발생했습니다.')
                        }
                    } else {
                        this.passwordCheck = false;
                    }
                } catch (error) {
                    alert('로그인 요청 실패')
                }
            }
        },
        goToGoogleLogin() {
            console.log("구현 예정")
        },
        async checkGoogleEmailDuplication(response) {
            console.log('구글 이메일 중복 검사')
            try {
                const googleEmail = response.email
                const isDuplicate = await this.requestEmailDuplicationCheckToDjango(googleEmail.trim())
                if (isDuplicate) {
                    const response = await this.requestAddRedisAccessTokenToDjango(googleEmail.trim())
                    if (response) {
                        alert('첫 화면으로 이동합니다!')
                        router.push('/')
                    }
                    else {
                        console.log("구글 isDuplicate 오류")
                    }
                } else {
                    await this.requestCreateNewSocialAccountToDjango(googleEmail.trim())
                }
            } catch (error) {
                console.log('이메일 중복 확인 실패', error)
                this.isEmailValid = false
            }
        },
        async handleGoogleLogin(googleResponse) {
            console.log("Google 응답 진입", googleResponse);
            if (googleResponse.credential) {
                const credential = googleResponse.credential;
                try {
                    console.log('Django 서버로 Google 인증 요청');
                    const response = await this.requestGoogleLoginToDjango({
                        credential: credential,
                        clientId: this.clientId
                    });
                    if (response) {
                        this.checkGoogleEmailDuplication(response)
                    }
                    else {
                        console.log("에러 발생")
                    }

                } catch (error) {
                    console.log('Google 로그인 요청 실패', error);
                }
            } else {
                console.log('Google 인증 정보를 찾을 수 없음');
            }
        },
        initializeGoogleSignIn() {
            if (typeof window.google !== 'undefined') {
                window.google.accounts.id.initialize({
                    client_id: this.clientId,
                    callback: this.handleGoogleLogin
                });
                window.google.accounts.id.prompt();
            } else {
                console.error('구글 API 설정 누락');
            }
        },
        forgotPassword() {
            console.log('비밀번호 찾기 기능 구현 예정');
        },
        toggleShopPopup(shop) {
            if (shop === 'forgotPassword' && !this.email) {
                alert('이메일을 먼저 입력해주세요.');
                return;
            }
            this.currentShop = this.currentShop === shop ? null : shop;
        },
        closeShopPopup() {
            this.currentShop = null;
            location.reload();
        },
    },
    mounted() {
        if (typeof window.google !== 'undefined') {
            this.initializeGoogleSignIn();
        } else {
            window.onload = () => {
                this.initializeGoogleSignIn();
            };
        }
    }
};
</script>

<style scoped>
.v-container {
    background-color: #F6F1EB;
}

.v-card {
    background-color: v-bind('colors.cardBackground') !important;
    height: 490px;
}

.submit-button {
    background-color: v-bind('colors.buttonBackground') !important;
    color: white !important;
    font-size: 16px !important;
    font-weight: normal !important;
    text-transform: none !important;
    letter-spacing: normal !important;
    height: 40px !important;
    width: 200px !important;
}

.submit-button:hover {
    background-color: v-bind('colors.buttonHoverBackground') !important;
}

.google-login-btn {
    background-image: url("@/assets/images/fixed/googleLogin.png");
    background-size: contain;
    background-repeat: no-repeat;
    background-position: center;
    height: 50px;
    width: 200px;
}

.security {
    color: v-bind('colors.errorText') !important;
    font-size: 12px !important;
}

.forgotPassword {
    cursor: pointer;
}
</style>
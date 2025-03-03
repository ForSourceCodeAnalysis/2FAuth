<script setup>
    import Form from '@/components/formElements/Form'
    import { useUserStore } from '@/stores/user'
    import { useNotifyStore } from '@/stores/notify'
    import { useAppSettingsStore } from '@/stores/appSettings'
    import { webauthnService } from '@/services/webauthn/webauthnService'

    const $2fauth = inject('2fauth')
    const router = useRouter()
    const user = useUserStore()
    const notify = useNotifyStore()
    const appSettings = useAppSettingsStore()
    const showWebauthnForm = user.preferences.useWebauthnOnly ? true : useStorage($2fauth.prefix + 'showWebauthnForm', false) 
    const form = reactive(new Form({
        email: '',
        password: ''
    }))
    const isBusy = ref(false)

    /**
     * Toggle the form between legacy and webauthn method
     */
    function toggleForm() {
        form.clear()
        showWebauthnForm.value = ! showWebauthnForm.value
    }

    /**
     * Sign in using the login/password form
     */
    function LegacysignIn(e) {
        notify.clear()

        form.post('/user/login', {returnError: true}).then(async (response) => {
            await user.loginAs({
                name: response.data.name,
                email: response.data.email,
                oauth_provider: response.data.oauth_provider,
                preferences: response.data.preferences,
                isAdmin: response.data.is_admin,
            })

            router.push({ name: 'accounts' })
        })
        .catch(error => {
            if( error.response.status === 401 ) {
                notify.alert({text: trans('auth.forms.authentication_failed'), duration: 10000 })
            }
            else if( error.response.status !== 422 ) {
                notify.error(error)
            }
        })
    }

    /**
     * Sign in using webauthn
     */
    function webauthnLogin() {
        notify.clear()
        form.clear()
        isBusy.value = true

        webauthnService.authenticate(form.email).then(async (response) => {
            await user.loginAs({
                name: response.data.name,
                email: response.data.email,
                oauth_provider: response.data.oauth_provider,
                preferences: response.data.preferences,
                isAdmin: response.data.is_admin,
            })

            router.push({ name: 'accounts' })
        })
        .catch(error => {
            if ('webauthn' in error) {
                if (error.name == 'is-warning') {
                    notify.warn({ text: trans(error.message) })
                }
                else notify.alert({ text: trans(error.message) })
            }
            else if( error.response.status === 401 ) {
                notify.alert({text: trans('auth.forms.authentication_failed'), duration: 10000 })
            }
            else if( error.response.status == 422 ) {
                form.errors.set(form.extractErrors(error.response))
            }
            else {
                notify.error(error)
            }
        })
        .finally(() => {
            isBusy.value = false
        })
    }

</script>

<template>
    <!-- webauthn authentication -->
    <FormWrapper v-if="showWebauthnForm" title="auth.forms.webauthn_login" punchline="auth.welcome_to_2fauth">
        <div class="field">
            {{ $t('auth.webauthn.use_security_device_to_sign_in') }}
        </div>
        <form id="frmWebauthnLogin" @submit.prevent="webauthnLogin" @keydown="form.onKeydown($event)">
            <FormField v-model="form.email" fieldName="email" :fieldError="form.errors.get('email')" inputType="email" label="auth.forms.email" autofocus />
            <FormButtons :isBusy="isBusy" caption="commons.continue" submitId="btnContinue"/>
        </form>
        <div class="nav-links">
            <p>
                {{ $t('auth.webauthn.lost_your_device') }}&nbsp;
                <RouterLink id="lnkRecoverAccount" :to="{ name: 'webauthn.lost' }" class="is-link">
                    {{ $t('auth.webauthn.recover_your_account') }}
                </RouterLink>
            </p>
            <p v-if="!user.preferences.useWebauthnOnly">{{ $t('auth.sign_in_using') }}&nbsp;
                <a id="lnkSignWithLegacy" role="button" class="is-link" @keyup.enter="toggleForm" @click="toggleForm" tabindex="0">
                    {{ $t('auth.login_and_password') }}
                </a>
            </p>
            <p v-if="appSettings.disableRegistration == false" class="mt-4">
                {{ $t('auth.forms.dont_have_account_yet') }}&nbsp;
                <RouterLink id="lnkRegister" :to="{ name: 'register' }" class="is-link">
                    {{ $t('auth.register') }}
                </RouterLink>
            </p>
            <div v-if="appSettings.enableSso && Object.values($2fauth.config.sso).includes(true)" class="columns mt-4 is-variable is-1">
                <div class="column is-narrow py-1">
                    {{ $t('auth.or_continue_with') }}
                </div>
                <div class="column py-1">
                    <a v-if="$2fauth.config.sso.openid" id="lnkSignWithOpenID" class="button is-link is-outlined is-small ml-2" href="/socialite/redirect/openid">
                        OpenID<FontAwesomeIcon class="ml-2" :icon="['fab', 'openid']" />
                    </a>
                    <a v-if="$2fauth.config.sso.github" id="lnkSignWithGithub" class="button is-link is-outlined is-small ml-2" href="/socialite/redirect/github">
                        Github<FontAwesomeIcon class="ml-2" :icon="['fab', 'github-alt']" />
                    </a>
                </div>
            </div>
        </div>
    </FormWrapper>
    <!-- login/password legacy form -->
    <FormWrapper v-else title="auth.forms.login" punchline="auth.welcome_to_2fauth">
        <div v-if="$2fauth.isDemoApp" class="notification is-info has-text-centered is-radiusless" v-html="$t('auth.forms.welcome_to_demo_app_use_those_credentials')" />
        <div v-if="$2fauth.isTestingApp" class="notification is-warning has-text-centered is-radiusless" v-html="$t('auth.forms.welcome_to_testing_app_use_those_credentials')" />
        <form id="frmLegacyLogin" @submit.prevent="LegacysignIn" @keydown="form.onKeydown($event)">
            <FormField v-model="form.email" fieldName="email" :fieldError="form.errors.get('email')" inputType="email" label="auth.forms.email" autofocus />
            <FormPasswordField v-model="form.password" fieldName="password" :fieldError="form.errors.get('password')" label="auth.forms.password" />
            <FormButtons :isBusy="form.isBusy" caption="auth.sign_in" submitId="btnSignIn"/>
        </form>
        <div class="nav-links">
            <p>{{ $t('auth.forms.forgot_your_password') }}&nbsp;
                <RouterLink id="lnkResetPwd" :to="{ name: 'password.request' }" class="is-link" :aria-label="$t('auth.forms.reset_your_password')">
                    {{ $t('auth.forms.request_password_reset') }}
                </RouterLink>
            </p>
            <p >{{ $t('auth.sign_in_using') }}&nbsp;
                <a id="lnkSignWithWebauthn" role="button" class="is-link" @keyup.enter="toggleForm" @click="toggleForm" tabindex="0" :aria-label="$t('auth.sign_in_using_security_device')">
                    {{ $t('auth.webauthn.security_device') }}
                </a>
            </p>
            <p v-if="appSettings.disableRegistration == false" class="mt-4">
                {{ $t('auth.forms.dont_have_account_yet') }}&nbsp;
                <RouterLink id="lnkRegister" :to="{ name: 'register' }" class="is-link">
                    {{ $t('auth.register') }}
                </RouterLink>
            </p>
            <div v-if="appSettings.enableSso && Object.values($2fauth.config.sso).includes(true)" class="columns mt-4 is-variable is-1">
                <div class="column is-narrow py-1">
                    {{ $t('auth.or_continue_with') }}
                </div>
                <div class="column py-1">
                    <a v-if="$2fauth.config.sso.openid" id="lnkSignWithOpenID" class="button is-link is-outlined is-small mr-2" href="/socialite/redirect/openid">
                        OpenID<FontAwesomeIcon class="ml-2" :icon="['fab', 'openid']" />
                    </a>
                    <a v-if="$2fauth.config.sso.github" id="lnkSignWithGithub" class="button is-link is-outlined is-small mr-2" href="/socialite/redirect/github">
                        Github<FontAwesomeIcon class="ml-2" :icon="['fab', 'github-alt']" />
                    </a>
                </div>
            </div>
        </div>
    </FormWrapper>
    <!-- footer -->
    <VueFooter/>
</template>

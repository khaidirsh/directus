<template>
	<form @submit.prevent="onSubmit">
		<v-input v-model="email" autofocus autocomplete="username" type="email" :placeholder="t('email')" />
		<v-input v-model="password" type="password" autocomplete="current-password" :placeholder="t('password')" />
		<vue-hcaptcha sitekey="e45cae2e-2906-45bf-abe7-9424392c31c6" @verify="onVerify"></vue-hcaptcha>

		<transition-expand>
			<v-input v-if="requiresTFA" v-model="otp" type="text" :placeholder="t('otp')" autofocus />
		</transition-expand>

		<v-notice v-if="error" type="warning">
			{{ errorFormatted }}
		</v-notice>
		<div class="buttons">
			<v-button type="submit" :loading="loggingIn" large :disabled="!hCaptchaVerified">{{ t('sign_in') }}</v-button>
			<router-link to="/reset-password" class="forgot-password">
				{{ t('forgot_password') }}
			</router-link>
		</div>
	</form>
</template>

<script lang="ts">
import VueHcaptcha from '@hcaptcha/vue3-hcaptcha';
import { useI18n } from 'vue-i18n';
import { defineComponent, ref, computed, watch, toRefs } from 'vue';
import { useRouter } from 'vue-router';
import { login } from '@/auth';
import { RequestError } from '@/api';
import { translateAPIError } from '@/lang';
import { useUserStore } from '@/stores/user';

type Credentials = {
	email: string;
	password: string;
	captchaToken: string;
	otp?: string;
};

export default defineComponent({
	components: { VueHcaptcha },
	props: {
		provider: {
			type: String,
			required: true,
		},
	},
	setup(props) {
		const { t } = useI18n();

		const router = useRouter();

		const { provider } = toRefs(props);
		const loggingIn = ref(false);
		const email = ref<string | null>(null);
		const password = ref<string | null>(null);
		const error = ref<RequestError | string | null>(null);
		const otp = ref<string | null>(null);
		const requiresTFA = ref(false);
		const userStore = useUserStore();
		const hCaptchaVerified = ref(false);
		const hCaptchaToken = ref<string | null>(null);
		const hCaptchaEKey = ref<string | null>(null);

		watch(email, () => {
			if (requiresTFA.value === true) requiresTFA.value = false;
		});

		watch(provider, () => {
			email.value = null;
			password.value = null;
			error.value = null;
			otp.value = null;
			requiresTFA.value = false;
		});

		const errorFormatted = computed(() => {
			// Show "Wrong username or password" for wrongly formatted emails as well
			if (error.value === 'INVALID_PAYLOAD') {
				return translateAPIError('INVALID_CREDENTIALS');
			}

			if (error.value) {
				return translateAPIError(error.value);
			}
			return null;
		});

		return {
			t,
			errorFormatted,
			error,
			email,
			password,
			onSubmit,
			loggingIn,
			translateAPIError,
			otp,
			requiresTFA,
			onVerify,
			hCaptchaVerified,
			hCaptchaToken,
			hCaptchaEKey,
		};

		async function onSubmit() {
			if (email.value === null || password.value === null || !hCaptchaVerified.value) return;

			try {
				loggingIn.value = true;

				const credentials: Credentials = {
					email: email.value,
					password: password.value,
					captchaToken: hCaptchaToken.value,
				};

				if (otp.value) {
					credentials.otp = otp.value;
				}

				await login({ provider: provider.value, credentials });

				const redirectQuery = router.currentRoute.value.query.redirect as string;

				// Stores are hydrated after login
				const lastPage = userStore.currentUser?.last_page;

				router.push(redirectQuery || lastPage || '/content');
			} catch (err: any) {
				if (err.response?.data?.errors?.[0]?.extensions?.code === 'INVALID_OTP' && requiresTFA.value === false) {
					requiresTFA.value = true;
				} else {
					error.value = err.response?.data?.errors?.[0]?.extensions?.code || err;
				}
			} finally {
				loggingIn.value = false;
			}
		}

		function onVerify(tokenStr: string, eKey: string) {
			hCaptchaVerified.value = true;
			hCaptchaToken.value = tokenStr;
			hCaptchaEKey.value = eKey;
		}
	},
});
</script>

<style lang="scss" scoped>
.v-input,
.v-notice {
	margin-bottom: 20px;
}

.buttons {
	display: flex;
	align-items: center;
	justify-content: space-between;
}

.forgot-password {
	color: var(--foreground-subdued);
	transition: color var(--fast) var(--transition);

	&:hover {
		color: var(--foreground-normal);
	}
}
</style>

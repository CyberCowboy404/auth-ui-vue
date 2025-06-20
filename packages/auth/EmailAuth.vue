<template>
  <form
    :id="`auth-${authView}`"
    @submit.prevent="handleSubmit"
    autoComplete="on"
    :style="{
      width: '100%'
    }"
  >
    <Container direction="vertical" gap="large" :appearance="appearance">
      <Container direction="vertical" gap="large" :appearance="appearance">
        <div>
          <Label htmlFor="email" :appearance="appearance">
            {{ labels?.email_label }}
          </Label>
          <Input
            id="email"
            type="email"
            name="email"
            v-model="email"
            :shape="appearance?.shape"
            :placeholder="labels?.email_input_placeholder"
            autoComplete="email"
            :appearance="appearance"
          />
        </div>
        <div>
          <Label htmlFor="password" :appearance="appearance">
            {{ labels?.password_label }}
          </Label>
          <Input
            id="password"
            type="password"
            name="password"
            v-model="password"
            :shape="appearance?.shape"
            :placeholder="labels?.password_input_placeholder"
            :autoComplete="
              authView === 'sign_in' ? 'current-password' : 'new-password'
            "
            :appearance="appearance"
          />
        </div>
      </Container>

      <BrandButton
        type="submit"
        :shape="appearance?.shape"
        :brand="appearance?.brand"
        :loading="isLoading"
        :appearance="appearance"
      >
        {{ isLoading ? labels?.loading_button_label : labels?.button_label }}
      </BrandButton>

      <template v-if="showLinks">
        <Container direction="vertical" gap="small" :appearance="appearance">
          <template v-if="authView === VIEWS.SIGN_IN && magicLink">
            <Anchor
              href="#auth-magic-link"
              :appearance="appearance"
              @click.prevent="setAuthView(VIEWS.MAGIC_LINK)"
            >
              {{ i18n?.magic_link?.link_text }}
            </Anchor>
          </template>
          <template v-if="authView === VIEWS.SIGN_IN">
            <Anchor
              href="#auth-forgot-password"
              :appearance="appearance"
              @click.prevent="setAuthView(VIEWS.FORGOTTEN_PASSWORD)"
            >
              {{ i18n?.forgotten_password?.link_text }}
            </Anchor>
          </template>
          <template v-if="authView === VIEWS.SIGN_IN">
            <Anchor
              href="#auth-sign-up"
              :appearance="appearance"
              @click.prevent="setAuthView(VIEWS.SIGN_UP)"
            >
              {{ i18n?.sign_up?.link_text }}
            </Anchor>
          </template>
          <template v-else>
            <Anchor
              href="#auth-sign-in"
              :appearance="appearance"
              @click.prevent="setAuthView(VIEWS.SIGN_IN)"
            >
              {{ i18n?.sign_in?.link_text }}
            </Anchor>
          </template>
        </Container>
      </template>
    </Container>

    <Message v-if="message" :appearance="appearance">{{ message }}</Message>

    <Message v-if="error" color="danger" :appearance="appearance">
      {{ error }}
    </Message>
  </form>
</template>

<script lang="ts" setup>
import { ref, computed, watch } from 'vue'
import { SupabaseClient } from '@supabase/supabase-js'
import { RedirectTo, VIEWS } from '@supabase/auth-ui-shared'

import {
  AuthViewKey,
  type Appearance,
  type AuthViewInjection,
  type AuthI18nVariables
} from '../types'
import {
  Anchor,
  BrandButton,
  Container,
  Input,
  Label,
  Message
} from '../ui/index'
import { injectStrict } from '../utils'

export interface EmailAuthProps {
  appearance?: Appearance
  supabaseClient: SupabaseClient
  defaultEmail?: string
  defaultPassword?: string
  redirectTo?: RedirectTo
  magicLink?: boolean
  showLinks?: boolean
  i18n?: AuthI18nVariables
  additionalData?: { [key: string]: any }
  beforeSubmit?: (email: string) => Promise<boolean | void> | boolean | void
}

const props = withDefaults(defineProps<EmailAuthProps>(), {})

const emit = defineEmits(['on-submit'])

const email = ref('')
const password = ref('')
const error = ref('')
const message = ref('')
const isLoading = ref(false)

const { authView, setAuthView } = injectStrict<AuthViewInjection>(AuthViewKey)

const labels = computed(
  () => props.i18n?.[authView.value] as AuthI18nVariables['sign_in' | 'sign_up']
)
const handleSubmit = async (e: Event) => {
  if (props.beforeSubmit) {
    const canSubmit = await props.beforeSubmit(email.value)
    if (canSubmit === false) {
      return
    }
  }

  // console.log(props)
  error.value = ''
  message.value = ''
  isLoading.value = true
  switch (authView.value) {
    case 'sign_in':
      emit('on-submit', email.value)
      const {
        // data: { user: signInUser, session: signInSession },
        error: signInError
      } = await props.supabaseClient.auth.signInWithPassword({
        email: email.value,
        password: password.value
      })
      if (signInError) {
        error.value = signInError.message
      }
      isLoading.value = false
      break
    case 'sign_up':
      let options: { emailRedirectTo: RedirectTo; data?: object } = {
        emailRedirectTo: props.redirectTo
      }
      if (props.additionalData) {
        options.data = props.additionalData
      }
      emit('on-submit', email.value)
      const {
        data: { user: signUpUser, session: signUpSession },
        error: signUpError
      } = await props.supabaseClient.auth.signUp({
        email: email.value,
        password: password.value,
        options
      })
      if (signUpError) error.value = signUpError.message
      // Check if session is null -> email confirmation setting is turned on
      else if (signUpUser && !signUpSession) {
        message.value = props.i18n?.sign_up?.confirmation_text as string
      }
      isLoading.value = false
    default:
      break
  }
}

watch(
  () => authView.value,
  () => {
    isLoading.value = false
  }
)
</script>

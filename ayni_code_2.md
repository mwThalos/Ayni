# Parte 2 — Layout, Landing y Auth

---

## `app/globals.css`
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  * { box-sizing: border-box; }
  body { @apply bg-background text-foreground antialiased; }
  ::-webkit-scrollbar { width: 5px; height: 5px; }
  ::-webkit-scrollbar-thumb { @apply bg-green-200 rounded-full; }
}

@keyframes fadeUp {
  from { opacity: 0; transform: translateY(12px); }
  to   { opacity: 1; transform: translateY(0); }
}
@keyframes slideIn {
  from { transform: translateX(100%); }
  to   { transform: translateX(0); }
}

.animate-fade-up { animation: fadeUp .3s ease; }
.animate-slide-in { animation: slideIn .25s ease; }
.card-hover { @apply transition-all duration-300 hover:-translate-y-1 hover:shadow-xl; }
```

---

## `app/layout.tsx`
```tsx
import type { Metadata } from 'next'
import { Inter } from 'next/font/google'
import './globals.css'
import { Toaster } from 'react-hot-toast'
import Navbar from '@/components/navbar'

const inter = Inter({ subsets: ['latin'] })

export const metadata: Metadata = {
  title: 'Ayni — Economía Circular',
  description: 'Red social de sostenibilidad y reciclaje en Pasto, Nariño',
  openGraph: {
    title: 'Ayni — Economía Circular',
    description: 'Comparte proyectos de reciclaje e intercambia materiales.',
    images: ['/og-image.png'],
  },
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="es">
      <body className={`${inter.className} min-h-screen bg-white text-gray-900`}>
        <Navbar />
        <main className="pt-14">{children}</main>
        <Toaster
          position="top-right"
          toastOptions={{
            style: { fontSize: 13, borderRadius: 10, fontWeight: 500 },
            success: { style: { background: '#dcfce7', color: '#14532d' } },
            error:   { style: { background: '#fee2e2', color: '#991b1b' } },
          }}
        />
      </body>
    </html>
  )
}
```

---

## `app/page.tsx` — Landing (¡ESTA ES LA QUE FALTABA!)
```tsx
import Link from 'next/link'
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export default async function LandingPage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (user) redirect('/feed')

  return (
    <div className="min-h-screen bg-white">
      {/* Hero */}
      <section className="max-w-4xl mx-auto px-6 pt-20 pb-16 text-center">
        <span className="inline-flex items-center gap-2 bg-green-50 text-green-700 text-xs font-semibold px-4 py-1.5 rounded-full mb-6 tracking-wide">
          🌿 ECONOMÍA CIRCULAR · PASTO, NARIÑO
        </span>
        <h1 className="text-5xl md:text-7xl font-extrabold text-gray-900 leading-tight mb-5 tracking-tight">
          Transforma residuos<br />
          <span className="text-green-600">en recursos</span>
        </h1>
        <p className="text-xl text-gray-500 max-w-xl mx-auto mb-10 leading-relaxed">
          Comparte proyectos de reciclaje, intercambia materiales con tu comunidad
          y descubre ideas creativas con inteligencia artificial.
        </p>
        <div className="flex gap-4 justify-center flex-wrap">
          <Link href="/register"
            className="bg-green-600 hover:bg-green-700 text-white px-8 py-3 rounded-xl text-base font-semibold transition shadow-lg shadow-green-200">
            Comenzar gratis
          </Link>
          <Link href="/feed"
            className="border-2 border-green-600 text-green-600 hover:bg-green-50 px-8 py-3 rounded-xl text-base font-semibold transition">
            Ver proyectos
          </Link>
        </div>
      </section>

      {/* Features */}
      <section className="max-w-5xl mx-auto px-6 py-16">
        <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-5">
          {[
            { icon: '⟳', title: 'Red Social',    desc: 'Comparte proyectos y conecta con recicladores de tu ciudad.' },
            { icon: '◈', title: 'Marketplace',   desc: 'Intercambia materiales reciclables con ubicación en el mapa.' },
            { icon: '✦', title: 'IA Creativa',   desc: 'Claude genera proyectos personalizados con tus materiales.' },
            { icon: '◎', title: 'Mensajería',    desc: 'Chatea directamente con otros miembros de la comunidad.' },
          ].map(f => (
            <div key={f.title} className="bg-gray-50 rounded-2xl p-6 card-hover">
              <div className="w-10 h-10 bg-green-100 rounded-xl flex items-center justify-center text-xl text-green-600 mb-4">
                {f.icon}
              </div>
              <h3 className="font-bold text-gray-900 mb-2">{f.title}</h3>
              <p className="text-sm text-gray-500 leading-relaxed">{f.desc}</p>
            </div>
          ))}
        </div>
      </section>

      <footer className="text-center text-xs text-gray-300 py-8 tracking-widest">
        AYNI · RECIPROCIDAD CON EL PLANETA
      </footer>
    </div>
  )
}
```

---

## `app/(auth)/login/page.tsx`
```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase/client'
import toast from 'react-hot-toast'

export default function LoginPage() {
  const [email,    setEmail]    = useState('')
  const [password, setPassword] = useState('')
  const [loading,  setLoading]  = useState(false)
  const router = useRouter()
  const supabase = createClient()

  const handleLogin = async () => {
    if (!email || password.length < 6) {
      toast.error('Ingresa un email válido y contraseña de 6+ caracteres')
      return
    }
    setLoading(true)
    const { error } = await supabase.auth.signInWithPassword({ email, password })
    if (error) { toast.error(error.message); setLoading(false); return }
    toast.success('¡Bienvenido!')
    router.push('/feed')
    router.refresh()
  }

  const handleGoogle = async () => {
    await supabase.auth.signInWithOAuth({
      provider: 'google',
      options: { redirectTo: `${window.location.origin}/auth/callback` },
    })
  }

  return (
    <div className="min-h-[80vh] flex items-center justify-center p-4">
      <div className="w-full max-w-sm bg-white border border-gray-100 rounded-2xl p-8 shadow-xl animate-fade-up">
        <div className="text-center mb-7">
          <div className="w-12 h-12 bg-green-600 rounded-xl flex items-center justify-center mx-auto mb-4 shadow-lg shadow-green-200">
            <span className="text-white text-xl">🌿</span>
          </div>
          <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">Bienvenido de vuelta</h1>
          <p className="text-sm text-gray-400 mt-1">Inicia sesión en tu cuenta</p>
        </div>

        <button onClick={handleGoogle}
          className="w-full flex items-center justify-center gap-3 border border-gray-200 rounded-xl py-2.5 text-sm font-medium hover:bg-gray-50 transition mb-5">
          <svg width="18" height="18" viewBox="0 0 24 24"><path fill="#4285F4" d="M23.745 12.27c0-.79-.07-1.54-.19-2.27h-11.3v4.51h6.47c-.29 1.48-1.14 2.73-2.4 3.58v3h3.86c2.26-2.09 3.56-5.17 3.56-8.82z"/><path fill="#34A853" d="M12.255 24c3.24 0 5.95-1.08 7.93-2.91l-3.86-3c-1.08.72-2.45 1.16-4.07 1.16-3.13 0-5.78-2.11-6.73-4.96h-3.98v3.09C3.515 21.3 7.565 24 12.255 24z"/><path fill="#FBBC05" d="M5.525 14.29c-.25-.72-.38-1.49-.38-2.29s.14-1.57.38-2.29V6.62h-3.98a11.86 11.86 0 0 0 0 10.76l3.98-3.09z"/><path fill="#EA4335" d="M12.255 4.75c1.77 0 3.35.61 4.6 1.8l3.42-3.42C18.205 1.19 15.495 0 12.255 0c-4.69 0-8.74 2.7-10.71 6.62l3.98 3.09c.95-2.85 3.6-4.96 6.73-4.96z"/></svg>
          Continuar con Google
        </button>

        <div className="relative mb-5">
          <div className="absolute inset-0 flex items-center"><div className="w-full border-t border-gray-100" /></div>
          <div className="relative flex justify-center text-xs text-gray-400"><span className="bg-white px-3">o con tu correo</span></div>
        </div>

        <div className="space-y-4">
          <div>
            <label className="text-xs font-semibold text-gray-500 uppercase tracking-wide block mb-1.5">Correo</label>
            <input
              type="email" placeholder="tu@correo.com" value={email}
              onChange={e => setEmail(e.target.value)}
              onKeyDown={e => e.key === 'Enter' && handleLogin()}
              className="w-full border border-gray-200 rounded-xl px-4 py-2.5 text-sm outline-none focus:border-green-400 transition"
            />
          </div>
          <div>
            <label className="text-xs font-semibold text-gray-500 uppercase tracking-wide block mb-1.5">Contraseña</label>
            <input
              type="password" placeholder="Mínimo 6 caracteres" value={password}
              onChange={e => setPassword(e.target.value)}
              onKeyDown={e => e.key === 'Enter' && handleLogin()}
              className="w-full border border-gray-200 rounded-xl px-4 py-2.5 text-sm outline-none focus:border-green-400 transition"
            />
          </div>
          <button onClick={handleLogin} disabled={loading}
            className="w-full bg-green-600 hover:bg-green-700 disabled:opacity-60 text-white py-2.5 rounded-xl text-sm font-semibold transition shadow-lg shadow-green-200">
            {loading ? 'Iniciando...' : 'Iniciar sesión'}
          </button>
        </div>

        <p className="text-center text-xs text-gray-400 mt-5">
          ¿No tienes cuenta?{' '}
          <Link href="/register" className="text-green-600 font-semibold hover:underline">Regístrate</Link>
        </p>
      </div>
    </div>
  )
}
```

---

## `app/(auth)/register/page.tsx`
```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase/client'
import toast from 'react-hot-toast'

const strength = (p: string) => {
  let s = 0
  if (p.length >= 8) s++
  if (/[A-Z]/.test(p)) s++
  if (/[0-9]/.test(p)) s++
  if (/[^A-Za-z0-9]/.test(p)) s++
  return s
}

export default function RegisterPage() {
  const [name,     setName]     = useState('')
  const [email,    setEmail]    = useState('')
  const [password, setPassword] = useState('')
  const [loading,  setLoading]  = useState(false)
  const router = useRouter()
  const supabase = createClient()

  const handleRegister = async () => {
    if (!name || !email || password.length < 6) {
      toast.error('Completa todos los campos (mínimo 6 caracteres)')
      return
    }
    setLoading(true)
    const { error } = await supabase.auth.signUp({
      email, password,
      options: { data: { full_name: name } },
    })
    if (error) { toast.error(error.message); setLoading(false); return }
    toast.success('¡Cuenta creada! Revisa tu correo.')
    router.push('/feed')
    router.refresh()
  }

  const handleGoogle = async () => {
    await supabase.auth.signInWithOAuth({
      provider: 'google',
      options: { redirectTo: `${window.location.origin}/auth/callback` },
    })
  }

  const s = strength(password)
  const sColor = ['', '#ef4444', '#f97316', '#eab308', '#22c55e'][s]
  const sLabel = ['', 'Muy débil', 'Débil', 'Media', 'Fuerte'][s]

  return (
    <div className="min-h-[80vh] flex items-center justify-center p-4">
      <div className="w-full max-w-sm bg-white border border-gray-100 rounded-2xl p-8 shadow-xl animate-fade-up">
        <div className="text-center mb-7">
          <div className="w-12 h-12 bg-green-600 rounded-xl flex items-center justify-center mx-auto mb-4 shadow-lg shadow-green-200">
            <span className="text-white text-xl">🌿</span>
          </div>
          <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">Únete a Ayni</h1>
          <p className="text-sm text-gray-400 mt-1">Crea tu cuenta gratis</p>
        </div>

        <button onClick={handleGoogle}
          className="w-full flex items-center justify-center gap-3 border border-gray-200 rounded-xl py-2.5 text-sm font-medium hover:bg-gray-50 transition mb-5">
          <svg width="18" height="18" viewBox="0 0 24 24"><path fill="#4285F4" d="M23.745 12.27c0-.79-.07-1.54-.19-2.27h-11.3v4.51h6.47c-.29 1.48-1.14 2.73-2.4 3.58v3h3.86c2.26-2.09 3.56-5.17 3.56-8.82z"/><path fill="#34A853" d="M12.255 24c3.24 0 5.95-1.08 7.93-2.91l-3.86-3c-1.08.72-2.45 1.16-4.07 1.16-3.13 0-5.78-2.11-6.73-4.96h-3.98v3.09C3.515 21.3 7.565 24 12.255 24z"/><path fill="#FBBC05" d="M5.525 14.29c-.25-.72-.38-1.49-.38-2.29s.14-1.57.38-2.29V6.62h-3.98a11.86 11.86 0 0 0 0 10.76l3.98-3.09z"/><path fill="#EA4335" d="M12.255 4.75c1.77 0 3.35.61 4.6 1.8l3.42-3.42C18.205 1.19 15.495 0 12.255 0c-4.69 0-8.74 2.7-10.71 6.62l3.98 3.09c.95-2.85 3.6-4.96 6.73-4.96z"/></svg>
          Continuar con Google
        </button>

        <div className="relative mb-5">
          <div className="absolute inset-0 flex items-center"><div className="w-full border-t border-gray-100" /></div>
          <div className="relative flex justify-center text-xs text-gray-400"><span className="bg-white px-3">o con tu correo</span></div>
        </div>

        <div className="space-y-4">
          {[
            { label: 'Nombre completo', val: name,  set: setName,     type: 'text',     ph: 'Tu nombre' },
            { label: 'Correo',          val: email, set: setEmail,    type: 'email',    ph: 'tu@correo.com' },
            { label: 'Contraseña',      val: password, set: setPassword, type: 'password', ph: 'Mínimo 6 caracteres' },
          ].map(f => (
            <div key={f.label}>
              <label className="text-xs font-semibold text-gray-500 uppercase tracking-wide block mb-1.5">{f.label}</label>
              <input
                type={f.type} placeholder={f.ph} value={f.val}
                onChange={e => f.set(e.target.value)}
                onKeyDown={e => e.key === 'Enter' && handleRegister()}
                className="w-full border border-gray-200 rounded-xl px-4 py-2.5 text-sm outline-none focus:border-green-400 transition"
              />
            </div>
          ))}

          {password.length > 0 && (
            <div>
              <div className="flex gap-1 mb-1">
                {[1,2,3,4].map(i => (
                  <div key={i} className="flex-1 h-1 rounded-full transition-all duration-300"
                    style={{ background: i <= s ? sColor : '#e5e7eb' }} />
                ))}
              </div>
              <span className="text-xs font-medium" style={{ color: sColor }}>{sLabel}</span>
            </div>
          )}

          <button onClick={handleRegister} disabled={loading}
            className="w-full bg-green-600 hover:bg-green-700 disabled:opacity-60 text-white py-2.5 rounded-xl text-sm font-semibold transition shadow-lg shadow-green-200">
            {loading ? 'Creando cuenta...' : 'Crear cuenta'}
          </button>
        </div>

        <p className="text-center text-xs text-gray-400 mt-5">
          ¿Ya tienes cuenta?{' '}
          <Link href="/login" className="text-green-600 font-semibold hover:underline">Inicia sesión</Link>
        </p>
      </div>
    </div>
  )
}
```

---

## `app/auth/callback/route.ts`
```ts
import { createClient } from '@/lib/supabase/server'
import { NextResponse } from 'next/server'

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url)
  const code = searchParams.get('code')
  const next = searchParams.get('next') ?? '/feed'

  if (code) {
    const supabase = await createClient()
    const { error } = await supabase.auth.exchangeCodeForSession(code)
    if (!error) return NextResponse.redirect(`${origin}${next}`)
  }

  return NextResponse.redirect(`${origin}/login?error=auth_error`)
}
```

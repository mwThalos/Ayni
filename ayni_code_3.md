# Parte 3 — Páginas principales

---

## `app/feed/page.tsx`
```tsx
import { createClient } from '@/lib/supabase/server'
import Link from 'next/link'
import Image from 'next/image'
import LikeButton from '@/components/like-button'
import SaveButton from '@/components/save-button'
import ShareButton from '@/components/share-button'

export default async function FeedPage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  const { data: posts } = await supabase
    .from('posts')
    .select('*, profiles(full_name, avatar_url)')
    .order('created_at', { ascending: false })
    .limit(60)

  const { data: userLikes } = user
    ? await supabase.from('likes').select('post_id').eq('user_id', user.id)
    : { data: [] }

  const { data: userSaves } = user
    ? await supabase.from('saves').select('post_id').eq('user_id', user.id)
    : { data: [] }

  const likedIds = new Set(userLikes?.map(l => l.post_id) || [])
  const savedIds = new Set(userSaves?.map(s => s.post_id) || [])

  return (
    <div className="max-w-7xl mx-auto px-4 py-8">
      <div className="flex items-center justify-between mb-8 flex-wrap gap-4">
        <div>
          <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">Explorar</h1>
          <p className="text-sm text-gray-400 mt-1">{posts?.length || 0} proyectos en la comunidad Ayni</p>
        </div>
        {user && (
          <Link href="/create"
            className="bg-green-600 hover:bg-green-700 text-white px-5 py-2.5 rounded-xl text-sm font-semibold transition shadow-lg shadow-green-200 flex items-center gap-2">
            + Publicar proyecto
          </Link>
        )}
      </div>

      {/* Pinterest grid */}
      <div className="columns-1 sm:columns-2 lg:columns-3 xl:columns-4 gap-4">
        {posts?.map((post: any) => (
          <div key={post.id} className="break-inside-avoid mb-4">
            <Link href={`/post/${post.id}`} className="block group">
              <div className="bg-white border border-gray-100 rounded-2xl overflow-hidden shadow-sm hover:-translate-y-1 hover:shadow-xl transition-all duration-300">
                {post.image_url && (
                  <div className="relative w-full">
                    <Image
                      src={post.image_url} alt={post.title}
                      width={400} height={300}
                      className="w-full object-cover"
                    />
                    <span className={`absolute top-2 right-2 text-xs font-bold px-2 py-1 rounded-full text-white ${post.type === 'material' ? 'bg-blue-500' : 'bg-green-600'}`}>
                      {post.type === 'material' ? 'Material' : 'Proyecto'}
                    </span>
                  </div>
                )}
                <div className="p-4">
                  <div className="flex items-center gap-2 mb-2">
                    <div className="w-6 h-6 rounded-full bg-green-100 flex items-center justify-center text-xs font-bold text-green-700 flex-shrink-0">
                      {post.profiles?.full_name?.[0]?.toUpperCase() || '?'}
                    </div>
                    <span className="text-xs text-gray-400 truncate">{post.profiles?.full_name}</span>
                  </div>
                  <h3 className="font-semibold text-gray-900 text-sm leading-snug mb-2 line-clamp-2">{post.title}</h3>
                  {post.description && (
                    <p className="text-xs text-gray-500 line-clamp-2 leading-relaxed mb-3">{post.description}</p>
                  )}
                  {post.materials_used?.length > 0 && (
                    <div className="flex flex-wrap gap-1 mb-3">
                      {post.materials_used.slice(0, 3).map((m: string) => (
                        <span key={m} className="text-xs bg-green-50 text-green-700 px-2 py-0.5 rounded-full">{m}</span>
                      ))}
                    </div>
                  )}
                  <div className="flex items-center gap-2 pt-2 border-t border-gray-50" onClick={e => e.preventDefault()}>
                    <LikeButton postId={post.id} initialLikes={post.likes_count} isLiked={likedIds.has(post.id)} userId={user?.id} />
                    <SaveButton postId={post.id} isSaved={savedIds.has(post.id)} userId={user?.id} />
                    <ShareButton postId={post.id} className="ml-auto" />
                  </div>
                </div>
              </div>
            </Link>
          </div>
        ))}
      </div>

      {!posts?.length && (
        <div className="text-center py-20 text-gray-400">
          <p className="text-5xl mb-4">🌱</p>
          <p className="text-lg font-medium">No hay publicaciones aún</p>
          <p className="text-sm mt-2">¡Sé el primero en compartir un proyecto!</p>
        </div>
      )}
    </div>
  )
}
```

---

## `app/post/[id]/page.tsx` — Detalle de publicación
```tsx
import { createClient } from '@/lib/supabase/server'
import { notFound } from 'next/navigation'
import Image from 'next/image'
import Link from 'next/link'
import CommentSection from '@/components/comment-section'
import ShareButton from '@/components/share-button'

export default async function PostDetailPage({ params }: { params: { id: string } }) {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  const { data: post } = await supabase
    .from('posts')
    .select('*, profiles(full_name, avatar_url, username)')
    .eq('id', params.id)
    .single()

  if (!post) return notFound()

  // Publicaciones similares
  const { data: related } = await supabase
    .from('posts')
    .select('id, title, image_url')
    .eq('type', post.type)
    .neq('id', post.id)
    .limit(4)

  const { data: comments } = await supabase
    .from('comments')
    .select('*, profiles(full_name, avatar_url)')
    .eq('post_id', post.id)
    .order('created_at', { ascending: true })

  return (
    <div className="max-w-3xl mx-auto px-4 py-8">
      <Link href="/feed" className="flex items-center gap-2 text-sm text-gray-400 hover:text-gray-700 mb-6 transition">
        ← Volver al feed
      </Link>

      {post.image_url && (
        <div className="relative w-full rounded-2xl overflow-hidden mb-6">
          <Image src={post.image_url} alt={post.title} width={800} height={500} className="w-full object-cover max-h-96" />
          <span className={`absolute top-3 right-3 text-xs font-bold px-3 py-1 rounded-full text-white ${post.type === 'material' ? 'bg-blue-500' : 'bg-green-600'}`}>
            {post.type === 'material' ? 'Material' : 'Proyecto'}
          </span>
        </div>
      )}

      <div className="flex items-center gap-3 mb-5">
        <div className="w-10 h-10 rounded-full bg-green-100 flex items-center justify-center font-bold text-green-700">
          {post.profiles?.full_name?.[0]?.toUpperCase()}
        </div>
        <div>
          <p className="font-semibold text-gray-900 text-sm">{post.profiles?.full_name}</p>
          <p className="text-xs text-gray-400">{new Date(post.created_at).toLocaleDateString('es-CO', { year:'numeric', month:'long', day:'numeric' })}</p>
        </div>
        <div className="ml-auto flex gap-2">
          <ShareButton postId={post.id} />
        </div>
      </div>

      <h1 className="text-3xl font-extrabold text-gray-900 mb-4 tracking-tight">{post.title}</h1>
      <p className="text-gray-600 leading-relaxed mb-6 text-base">{post.description}</p>

      {post.materials_used?.length > 0 && (
        <div className="mb-6">
          <h2 className="text-sm font-bold text-gray-500 uppercase tracking-wider mb-3">Materiales</h2>
          <div className="flex flex-wrap gap-2">
            {post.materials_used.map((m: string) => (
              <span key={m} className="bg-green-50 text-green-700 text-sm px-3 py-1 rounded-full">{m}</span>
            ))}
          </div>
        </div>
      )}

      {post.steps?.length > 0 && (
        <div className="mb-6">
          <h2 className="text-sm font-bold text-gray-500 uppercase tracking-wider mb-4">Paso a paso</h2>
          <div className="space-y-3">
            {post.steps.map((step: string, i: number) => (
              <div key={i} className="flex gap-4">
                <div className="w-7 h-7 rounded-full bg-green-600 text-white flex items-center justify-center text-xs font-bold flex-shrink-0">{i+1}</div>
                <p className="text-gray-700 text-sm leading-relaxed pt-1">{step}</p>
              </div>
            ))}
          </div>
        </div>
      )}

      {post.type === 'material' && post.location && (
        <div className="bg-blue-50 rounded-xl p-4 mb-6 flex items-center justify-between">
          <div>
            <p className="text-xs font-bold text-blue-600 uppercase tracking-wide mb-1">Ubicación</p>
            <p className="text-blue-800 font-semibold">{post.location}</p>
            {post.quantity && <p className="text-blue-600 text-sm mt-0.5">Disponible: {post.quantity}</p>}
          </div>
          <a href={`https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(post.location)}`}
            target="_blank" rel="noopener noreferrer"
            className="bg-blue-600 text-white text-xs font-semibold px-4 py-2 rounded-xl hover:bg-blue-700 transition">
            Ver en mapa
          </a>
        </div>
      )}

      {/* Comentarios */}
      <CommentSection postId={post.id} initialComments={comments || []} currentUser={user} />

      {/* Relacionados */}
      {related && related.length > 0 && (
        <div className="mt-10">
          <h2 className="text-sm font-bold text-gray-500 uppercase tracking-wider mb-4">Proyectos similares</h2>
          <div className="grid grid-cols-2 sm:grid-cols-4 gap-3">
            {related.map((r: any) => (
              <Link key={r.id} href={`/post/${r.id}`}
                className="rounded-xl overflow-hidden border border-gray-100 hover:shadow-lg transition-all hover:-translate-y-1">
                {r.image_url && <Image src={r.image_url} alt={r.title} width={200} height={120} className="w-full h-24 object-cover" />}
                <p className="text-xs font-medium p-2 text-gray-700 line-clamp-2">{r.title}</p>
              </Link>
            ))}
          </div>
        </div>
      )}
    </div>
  )
}
```

---

## `app/create/page.tsx`
```tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import { createClient } from '@/lib/supabase/client'
import toast from 'react-hot-toast'

export default function CreatePage() {
  const [type,    setType]    = useState<'project'|'material'>('project')
  const [title,   setTitle]   = useState('')
  const [desc,    setDesc]    = useState('')
  const [mats,    setMats]    = useState<string[]>([])
  const [matIn,   setMatIn]   = useState('')
  const [steps,   setSteps]   = useState([''])
  const [qty,     setQty]     = useState('')
  const [loc,     setLoc]     = useState('')
  const [image,   setImage]   = useState<File | null>(null)
  const [preview, setPreview] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)
  const router = useRouter()
  const supabase = createClient()

  const handleImage = (e: React.ChangeEvent<HTMLInputElement>) => {
    const f = e.target.files?.[0]
    if (!f) return
    setImage(f)
    setPreview(URL.createObjectURL(f))
  }

  const moderate = async () => {
    try {
      const res = await fetch('/api/ai/moderate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, description: desc }),
      })
      const data = await res.json()
      return data.ok !== false
    } catch { return true }
  }

  const handleSubmit = async () => {
    if (!title.trim()) { toast.error('El título es obligatorio'); return }
    setLoading(true)

    const { data: { user } } = await supabase.auth.getUser()
    if (!user) { toast.error('Debes iniciar sesión'); setLoading(false); return }

    // Moderación con IA
    const ok = await moderate()
    if (!ok) { toast.error('El contenido no cumple las normas de la comunidad'); setLoading(false); return }

    let imageUrl = ''
    if (image) {
      const ext = image.name.split('.').pop()
      const { data, error } = await supabase.storage
        .from('posts')
        .upload(`${user.id}/${Date.now()}.${ext}`, image)
      if (error) { toast.error('Error subiendo imagen'); setLoading(false); return }
      const { data: { publicUrl } } = supabase.storage.from('posts').getPublicUrl(data.path)
      imageUrl = publicUrl
    }

    const { error } = await supabase.from('posts').insert({
      user_id: user.id,
      type,
      title,
      description: desc,
      image_url: imageUrl,
      materials_used: mats,
      steps: steps.filter(s => s.trim()),
      quantity: qty || null,
      location: loc || null,
    })

    if (error) { toast.error(error.message); setLoading(false); return }
    toast.success('¡Publicación creada!')
    router.push('/feed')
    router.refresh()
  }

  const addMat = () => {
    if (matIn.trim() && !mats.includes(matIn.trim())) {
      setMats([...mats, matIn.trim()])
      setMatIn('')
    }
  }

  const inp = "w-full border border-gray-200 rounded-xl px-4 py-2.5 text-sm outline-none focus:border-green-400 transition"
  const lbl = "text-xs font-semibold text-gray-500 uppercase tracking-wide block mb-1.5"

  return (
    <div className="max-w-2xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-extrabold text-gray-900 mb-1 tracking-tight">Nueva publicación</h1>
      <p className="text-sm text-gray-400 mb-6">El contenido es revisado por IA antes de publicarse</p>

      <div className="space-y-5">
        {/* Tipo */}
        <div>
          <label className={lbl}>Tipo de publicación</label>
          <div className="flex gap-3">
            {[['project','♻ Proyecto de reciclaje'],['material','◈ Material disponible']].map(([v, l]) => (
              <button key={v} onClick={() => setType(v as any)}
                className={`flex-1 py-2.5 rounded-xl border-2 text-sm font-semibold transition ${type === v ? 'border-green-600 bg-green-50 text-green-700' : 'border-gray-200 text-gray-500 hover:border-gray-300'}`}>
                {l}
              </button>
            ))}
          </div>
        </div>

        {/* Imagen */}
        <div className={`rounded-2xl border-2 overflow-hidden transition ${preview ? 'border-gray-200' : 'border-dashed border-green-300 hover:border-green-500'}`}>
          {preview ? (
            <div className="relative">
              <img src={preview} alt="preview" className="w-full max-h-64 object-cover" />
              <button onClick={() => { setImage(null); setPreview(null) }}
                className="absolute top-2 right-2 bg-white rounded-full w-7 h-7 flex items-center justify-center shadow text-gray-600 hover:text-red-500 transition text-lg">×</button>
            </div>
          ) : (
            <label className="flex flex-col items-center justify-center py-10 cursor-pointer">
              <span className="text-3xl mb-2">📷</span>
              <span className="text-sm font-semibold text-green-600">Subir imagen o video</span>
              <span className="text-xs text-gray-400 mt-1">PNG, JPG, MP4 — hasta 10MB</span>
              <input type="file" accept="image/*,video/*" className="hidden" onChange={handleImage} />
            </label>
          )}
        </div>

        <div><label className={lbl}>Título *</label><input className={inp} value={title} onChange={e => setTitle(e.target.value)} placeholder={type === 'project' ? 'Nombre del proyecto' : 'Nombre del material'} /></div>
        <div><label className={lbl}>Descripción</label><textarea className={`${inp} min-h-[80px] resize-none`} value={desc} onChange={e => setDesc(e.target.value)} placeholder="Describe tu publicación..." /></div>

        {type === 'material' && (
          <div className="grid grid-cols-2 gap-4">
            <div><label className={lbl}>Cantidad</label><input className={inp} value={qty} onChange={e => setQty(e.target.value)} placeholder="Ej: 20 unidades" /></div>
            <div><label className={lbl}>Ubicación</label><input className={inp} value={loc} onChange={e => setLoc(e.target.value)} placeholder="Barrio / sector" /></div>
          </div>
        )}

        {/* Materiales */}
        <div>
          <label className={lbl}>{type === 'project' ? 'Materiales utilizados' : 'Tipo de material'}</label>
          <div className="flex gap-2">
            <input className={`${inp} flex-1`} value={matIn} onChange={e => setMatIn(e.target.value)}
              placeholder="Añadir material..." onKeyDown={e => { if (e.key === 'Enter') { e.preventDefault(); addMat() } }} />
            <button onClick={addMat} className="bg-green-600 text-white px-4 rounded-xl text-sm font-semibold hover:bg-green-700 transition">+</button>
          </div>
          <div className="flex flex-wrap gap-2 mt-2">
            {mats.map((m, i) => (
              <span key={i} className="bg-green-50 text-green-700 text-xs px-3 py-1 rounded-full flex items-center gap-1.5">
                {m}<button onClick={() => setMats(ms => ms.filter((_,j) => j !== i))} className="text-green-500 hover:text-red-500 transition text-sm leading-none">×</button>
              </span>
            ))}
          </div>
        </div>

        {/* Pasos */}
        {type === 'project' && (
          <div>
            <label className={lbl}>Pasos del proceso</label>
            <div className="space-y-2">
              {steps.map((s, i) => (
                <div key={i} className="flex gap-2 items-center">
                  <span className="text-green-600 font-bold text-xs w-5 flex-shrink-0">{i+1}.</span>
                  <input className={`${inp} flex-1`} value={s} placeholder={`Paso ${i+1}...`}
                    onChange={e => { const n = [...steps]; n[i] = e.target.value; setSteps(n) }} />
                  {steps.length > 1 && (
                    <button onClick={() => setSteps(ss => ss.filter((_,j) => j !== i))} className="text-gray-300 hover:text-red-400 transition text-lg">×</button>
                  )}
                </div>
              ))}
            </div>
            <button onClick={() => setSteps(ss => [...ss, ''])} className="text-green-600 text-xs font-semibold mt-2 hover:text-green-800 transition">+ Añadir paso</button>
          </div>
        )}

        <button onClick={handleSubmit} disabled={loading}
          className="w-full bg-green-600 hover:bg-green-700 disabled:opacity-60 text-white py-3 rounded-xl font-semibold text-sm transition shadow-lg shadow-green-200">
          {loading ? 'Publicando...' : '🌱 Publicar'}
        </button>
      </div>
    </div>
  )
}
```

---

## `app/marketplace/page.tsx`
```tsx
import { createClient } from '@/lib/supabase/server'
import Link from 'next/link'
import Image from 'next/image'

export default async function MarketplacePage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  const { data: materials } = await supabase
    .from('posts')
    .select('*, profiles(full_name)')
    .eq('type', 'material')
    .order('created_at', { ascending: false })

  return (
    <div className="max-w-7xl mx-auto px-4 py-8">
      <div className="flex items-center justify-between mb-8 flex-wrap gap-4">
        <div>
          <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">Materiales disponibles</h1>
          <p className="text-sm text-gray-400 mt-1">{materials?.length || 0} materiales en tu comunidad</p>
        </div>
        {user && (
          <Link href="/create?type=material"
            className="bg-blue-600 hover:bg-blue-700 text-white px-5 py-2.5 rounded-xl text-sm font-semibold transition shadow-lg shadow-blue-100 flex items-center gap-2">
            + Publicar material
          </Link>
        )}
      </div>
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-4">
        {materials?.map((m: any) => (
          <div key={m.id} className="bg-white border border-gray-100 rounded-2xl overflow-hidden shadow-sm hover:-translate-y-1 hover:shadow-xl transition-all duration-300">
            {m.image_url && (
              <Image src={m.image_url} alt={m.title} width={400} height={200} className="w-full h-40 object-cover" />
            )}
            <div className="p-4">
              <h3 className="font-semibold text-gray-900 text-sm mb-1">{m.title}</h3>
              <p className="text-xs text-gray-500 line-clamp-2 mb-3 leading-relaxed">{m.description}</p>
              {(m.quantity || m.location) && (
                <div className="text-xs text-gray-400 mb-3 flex gap-3">
                  {m.quantity && <span>{m.quantity}</span>}
                  {m.location && <span>📍{m.location}</span>}
                </div>
              )}
              <div className="flex gap-2">
                {m.location && (
                  <a href={`https://www.google.com/maps/search/?api=1&query=${encodeURIComponent(m.location)}`}
                    target="_blank" rel="noopener noreferrer"
                    className="flex-1 bg-blue-50 text-blue-700 text-xs font-semibold py-2 rounded-xl text-center hover:bg-blue-100 transition">
                    Ver en mapa
                  </a>
                )}
                <Link href={`/post/${m.id}`}
                  className="flex-1 bg-gray-50 text-gray-600 text-xs font-semibold py-2 rounded-xl text-center hover:bg-gray-100 transition">
                  Ver detalle
                </Link>
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## `app/ai/page.tsx`
```tsx
'use client'

import { useState } from 'react'
import { useRouter } from 'next/navigation'
import toast from 'react-hot-toast'

interface Idea {
  titulo: string; descripcion: string
  materiales: string[]; pasos: string[]
  impacto: string; dificultad: string
}

export default function AIPage() {
  const [input,   setInput]   = useState('')
  const [ideas,   setIdeas]   = useState<Idea[]>([])
  const [loading, setLoading] = useState(false)
  const router = useRouter()

  const generate = async () => {
    if (!input.trim()) return
    setLoading(true); setIdeas([])
    try {
      const res = await fetch('/api/ai/ideas', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ materials: input }),
      })
      const data = await res.json()
      setIdeas(data.ideas || [])
    } catch { toast.error('Error al generar ideas') }
    setLoading(false)
  }

  const diffColor: Record<string,string> = {
    'Fácil': 'bg-green-50 text-green-700',
    'Medio': 'bg-yellow-50 text-yellow-700',
    'Difícil': 'bg-red-50 text-red-700',
  }

  return (
    <div className="max-w-3xl mx-auto px-4 py-8">
      <div className="text-center mb-10">
        <div className="w-14 h-14 bg-gradient-to-br from-green-500 to-emerald-700 rounded-2xl flex items-center justify-center mx-auto mb-4 shadow-xl shadow-green-200">
          <span className="text-2xl">✦</span>
        </div>
        <h1 className="text-3xl font-extrabold text-gray-900 tracking-tight mb-3">Ideas con IA</h1>
        <p className="text-gray-500 max-w-md mx-auto leading-relaxed">
          Describe los materiales que tienes y Claude generará proyectos creativos de reutilización para Pasto, Nariño.
        </p>
      </div>

      <div className="bg-white border border-gray-100 rounded-2xl p-6 shadow-sm mb-8">
        <label className="text-xs font-bold text-gray-500 uppercase tracking-wider block mb-3">¿Qué materiales tienes?</label>
        <textarea
          className="w-full border border-gray-200 rounded-xl px-4 py-3 text-sm outline-none focus:border-green-400 transition resize-none min-h-[90px] mb-4"
          placeholder="Ej: botellas PET, cartón corrugado, tapas de botella, pallets de madera..."
          value={input} onChange={e => setInput(e.target.value)}
        />
        <button onClick={generate} disabled={loading || !input.trim()}
          className="w-full bg-green-600 hover:bg-green-700 disabled:opacity-50 text-white py-3 rounded-xl font-semibold text-sm transition shadow-lg shadow-green-200 flex items-center justify-center gap-2">
          {loading
            ? <><span className="inline-block w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"/>Generando...</>
            : '✦ Generar ideas con IA'}
        </button>
      </div>

      {ideas.length > 0 && (
        <div className="space-y-4 animate-fade-up">
          <h2 className="font-bold text-gray-800">{ideas.length} ideas para ti</h2>
          {ideas.map((idea, i) => (
            <div key={i} className="bg-white border border-gray-100 rounded-2xl overflow-hidden shadow-sm">
              <div className="h-1 bg-gradient-to-r from-green-500 to-emerald-600" />
              <div className="p-6">
                <div className="flex items-start justify-between mb-3">
                  <h3 className="font-bold text-gray-900 text-lg">{idea.titulo}</h3>
                  {idea.dificultad && (
                    <span className={`text-xs font-bold px-2 py-1 rounded-full ml-3 flex-shrink-0 ${diffColor[idea.dificultad] || 'bg-gray-50 text-gray-600'}`}>
                      {idea.dificultad}
                    </span>
                  )}
                </div>
                <p className="text-gray-600 text-sm leading-relaxed mb-4">{idea.descripcion}</p>

                {idea.materiales?.length > 0 && (
                  <div className="mb-4">
                    <p className="text-xs font-bold text-gray-500 uppercase tracking-wide mb-2">Materiales</p>
                    <div className="flex flex-wrap gap-1.5">
                      {idea.materiales.map((m,j) => <span key={j} className="text-xs bg-green-50 text-green-700 px-2.5 py-1 rounded-full">{m}</span>)}
                    </div>
                  </div>
                )}

                {idea.pasos?.length > 0 && (
                  <div className="mb-4">
                    <p className="text-xs font-bold text-gray-500 uppercase tracking-wide mb-2">Pasos</p>
                    <div className="space-y-1.5">
                      {idea.pasos.map((p,j) => (
                        <div key={j} className="flex gap-2 text-sm text-gray-700">
                          <span className="text-green-600 font-bold flex-shrink-0">{j+1}.</span>{p}
                        </div>
                      ))}
                    </div>
                  </div>
                )}

                {idea.impacto && (
                  <div className="bg-green-50 rounded-xl p-3 mb-4">
                    <p className="text-xs font-bold text-green-600 mb-1">🌍 Impacto ambiental</p>
                    <p className="text-sm text-green-800">{idea.impacto}</p>
                  </div>
                )}

                <button
                  onClick={() => { sessionStorage.setItem('ayni_prefill', JSON.stringify(idea)); router.push('/create') }}
                  className="text-green-600 border border-green-200 hover:bg-green-50 text-xs font-semibold px-4 py-2 rounded-xl transition">
                  Publicar este proyecto →
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  )
}
```

---

## `app/profile/page.tsx`
```tsx
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'
import Link from 'next/link'
import Image from 'next/image'

export default async function ProfilePage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) redirect('/login')

  const { data: profile } = await supabase.from('profiles').select('*').eq('id', user.id).single()
  const { data: myPosts  } = await supabase.from('posts').select('*').eq('user_id', user.id).order('created_at', { ascending: false })
  const { data: mySaves  } = await supabase.from('saves').select('post_id, posts(*)').eq('user_id', user.id)
  const savedPosts = mySaves?.map((s: any) => s.posts).filter(Boolean) || []

  const stats = [
    { label: 'Publicaciones', value: myPosts?.length || 0 },
    { label: 'Guardados',     value: savedPosts.length },
    { label: 'CO₂ ahorrado',  value: `${profile?.co2_saved || 0} kg` },
  ]

  return (
    <div className="max-w-4xl mx-auto px-4 py-8">
      {/* Header */}
      <div className="bg-gradient-to-br from-green-50 to-white border border-green-100 rounded-2xl p-8 mb-8">
        <div className="flex items-start gap-6 flex-wrap">
          <div className="w-20 h-20 rounded-full bg-green-100 overflow-hidden flex items-center justify-center border-4 border-white shadow-lg flex-shrink-0">
            {profile?.avatar_url
              ? <Image src={profile.avatar_url} alt="avatar" width={80} height={80} className="object-cover w-full h-full" />
              : <span className="text-3xl font-bold text-green-600">{profile?.full_name?.[0]?.toUpperCase() || '?'}</span>}
          </div>
          <div className="flex-1">
            <h1 className="text-2xl font-extrabold text-gray-900 tracking-tight">{profile?.full_name || 'Usuario'}</h1>
            <p className="text-gray-400 text-sm mb-2">{user.email}</p>
            <p className="text-gray-600 text-sm">{profile?.bio || 'Sin biografía.'}</p>
          </div>
          <form action="/api/auth/signout" method="post">
            <button className="text-sm text-red-500 border border-red-200 hover:bg-red-50 px-4 py-2 rounded-xl transition">Cerrar sesión</button>
          </form>
        </div>
        <div className="grid grid-cols-3 gap-4 mt-6">
          {stats.map(s => (
            <div key={s.label} className="bg-white rounded-xl p-4 text-center border border-gray-100 shadow-sm">
              <p className="text-2xl font-extrabold text-green-600">{s.value}</p>
              <p className="text-xs text-gray-400 mt-1">{s.label}</p>
            </div>
          ))}
        </div>
      </div>

      {/* Posts */}
      <h2 className="font-bold text-gray-800 mb-4">Mis publicaciones</h2>
      {myPosts && myPosts.length > 0 ? (
        <div className="columns-1 sm:columns-2 lg:columns-3 gap-4">
          {myPosts.map((post: any) => (
            <Link key={post.id} href={`/post/${post.id}`} className="block mb-4 break-inside-avoid">
              <div className="bg-white border border-gray-100 rounded-2xl overflow-hidden hover:shadow-lg transition-all hover:-translate-y-1">
                {post.image_url && <Image src={post.image_url} alt={post.title} width={400} height={300} className="w-full object-cover" />}
                <div className="p-3">
                  <p className="font-semibold text-sm text-gray-900 line-clamp-2">{post.title}</p>
                </div>
              </div>
            </Link>
          ))}
        </div>
      ) : (
        <div className="text-center py-12 border-2 border-dashed border-gray-200 rounded-2xl">
          <p className="text-gray-400 mb-3">Aún no has publicado nada</p>
          <Link href="/create" className="bg-green-600 text-white text-sm font-semibold px-5 py-2 rounded-xl hover:bg-green-700 transition">Primera publicación</Link>
        </div>
      )}

      {/* Guardados */}
      {savedPosts.length > 0 && (
        <>
          <h2 className="font-bold text-gray-800 mt-10 mb-4">Guardados para después</h2>
          <div className="columns-1 sm:columns-2 lg:columns-3 gap-4">
            {savedPosts.map((post: any) => (
              <Link key={post.id} href={`/post/${post.id}`} className="block mb-4 break-inside-avoid">
                <div className="bg-white border border-gray-100 rounded-2xl overflow-hidden hover:shadow-lg transition-all hover:-translate-y-1">
                  {post.image_url && <Image src={post.image_url} alt={post.title} width={400} height={300} className="w-full object-cover" />}
                  <div className="p-3"><p className="font-semibold text-sm text-gray-900 line-clamp-2">{post.title}</p></div>
                </div>
              </Link>
            ))}
          </div>
        </>
      )}
    </div>
  )
}
```

---

## `app/messages/page.tsx`
```tsx
'use client'

import { useEffect, useState, useRef } from 'react'
import { createClient } from '@/lib/supabase/client'

export default function MessagesPage() {
  const [conversations, setConversations] = useState<any[]>([])
  const [active,        setActive]        = useState<string | null>(null)
  const [messages,      setMessages]      = useState<any[]>([])
  const [text,          setText]          = useState('')
  const [user,          setUser]          = useState<any>(null)
  const bottomRef = useRef<HTMLDivElement>(null)
  const supabase  = createClient()

  useEffect(() => {
    supabase.auth.getUser().then(({ data: { user } }) => {
      setUser(user)
      if (user) loadConversations(user.id)
    })
  }, [])

  const loadConversations = async (uid: string) => {
    const { data } = await supabase
      .from('messages')
      .select('*, sender:profiles!sender_id(id,full_name), receiver:profiles!receiver_id(id,full_name)')
      .or(`sender_id.eq.${uid},receiver_id.eq.${uid}`)
      .order('created_at', { ascending: false })

    // Agrupar por conversación
    const seen = new Set()
    const convs: any[] = []
    data?.forEach(m => {
      const other = m.sender_id === uid ? m.receiver : m.sender
      if (!seen.has(other.id)) {
        seen.add(other.id)
        convs.push({ other, lastMsg: m })
      }
    })
    setConversations(convs)
  }

  const openConversation = async (otherId: string) => {
    setActive(otherId)
    const { data } = await supabase
      .from('messages')
      .select('*, sender:profiles!sender_id(full_name)')
      .or(`and(sender_id.eq.${user.id},receiver_id.eq.${otherId}),and(sender_id.eq.${otherId},receiver_id.eq.${user.id})`)
      .order('created_at', { ascending: true })
    setMessages(data || [])

    // Marcar como leídos
    await supabase.from('messages')
      .update({ read: true })
      .eq('sender_id', otherId)
      .eq('receiver_id', user.id)

    // Suscribir a nuevos mensajes en tiempo real
    supabase.channel(`conv-${otherId}`)
      .on('postgres_changes', { event: 'INSERT', schema: 'public', table: 'messages' }, payload => {
        setMessages(prev => [...prev, payload.new])
        setTimeout(() => bottomRef.current?.scrollIntoView({ behavior: 'smooth' }), 100)
      })
      .subscribe()
  }

  const sendMessage = async () => {
    if (!text.trim() || !active) return
    await supabase.from('messages').insert({ sender_id: user.id, receiver_id: active, content: text })
    setText('')
    loadConversations(user.id)
  }

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  if (!user) return <div className="text-center py-20 text-gray-400">Inicia sesión para ver tus mensajes</div>

  return (
    <div className="max-w-4xl mx-auto px-4 py-8">
      <h1 className="text-2xl font-extrabold text-gray-900 mb-6 tracking-tight">Mensajes</h1>
      <div className="bg-white border border-gray-100 rounded-2xl shadow-sm flex overflow-hidden" style={{ height: '70vh' }}>

        {/* Lista conversaciones */}
        <div className="w-64 border-r border-gray-100 overflow-y-auto flex-shrink-0">
          {conversations.length === 0 && (
            <p className="text-xs text-gray-400 text-center p-6">No tienes conversaciones aún</p>
          )}
          {conversations.map(c => (
            <button key={c.other.id} onClick={() => openConversation(c.other.id)}
              className={`w-full flex items-center gap-3 p-4 text-left hover:bg-gray-50 transition border-l-2 ${active === c.other.id ? 'border-green-600 bg-green-50' : 'border-transparent'}`}>
              <div className="w-9 h-9 rounded-full bg-green-100 flex items-center justify-center font-bold text-green-700 text-sm flex-shrink-0">
                {c.other.full_name?.[0]?.toUpperCase()}
              </div>
              <div className="min-w-0">
                <p className="font-semibold text-sm text-gray-900 truncate">{c.other.full_name}</p>
                <p className="text-xs text-gray-400 truncate">{c.lastMsg.content}</p>
              </div>
            </button>
          ))}
        </div>

        {/* Chat */}
        {active ? (
          <div className="flex-1 flex flex-col min-w-0">
            <div className="flex-1 overflow-y-auto p-4 space-y-3">
              {messages.map((m: any) => {
                const mine = m.sender_id === user.id
                return (
                  <div key={m.id} className={`flex ${mine ? 'justify-end' : 'justify-start'}`}>
                    <div className={`max-w-xs px-4 py-2.5 rounded-2xl text-sm ${mine ? 'bg-green-600 text-white' : 'bg-gray-100 text-gray-800'}`}>
                      {m.content}
                    </div>
                  </div>
                )
              })}
              <div ref={bottomRef} />
            </div>
            <div className="flex gap-2 p-4 border-t border-gray-100">
              <input
                className="flex-1 border border-gray-200 rounded-xl px-4 py-2.5 text-sm outline-none focus:border-green-400 transition"
                placeholder="Escribe un mensaje..." value={text}
                onChange={e => setText(e.target.value)}
                onKeyDown={e => e.key === 'Enter' && sendMessage()}
              />
              <button onClick={sendMessage}
                className="bg-green-600 text-white px-5 py-2.5 rounded-xl text-sm font-semibold hover:bg-green-700 transition">
                Enviar
              </button>
            </div>
          </div>
        ) : (
          <div className="flex-1 flex items-center justify-center text-gray-400 flex-col gap-2">
            <span className="text-4xl">💬</span>
            <p className="text-sm">Selecciona una conversación</p>
          </div>
        )}
      </div>
    </div>
  )
}
```

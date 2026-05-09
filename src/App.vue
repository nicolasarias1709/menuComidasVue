<template>
  <div id="app">

    <!-- SPINNER -->
    <transition name="fade">
      <div v-if="loading" class="spinner-overlay">
        <div class="spinner"></div>
        <div class="spinner-label">{{ loadingText }}</div>
      </div>
    </transition>

    <!-- NOTIFICACIONES -->
    <div class="notifs">
      <transition-group name="notif">
        <div v-for="n in notifs" :key="n.id" :class="['notif', n.type]">
          <span>{{ n.icon }}</span><span>{{ n.text }}</span>
        </div>
      </transition-group>
    </div>

    <!-- HEADER -->
    <header>
      <div class="header-inner">
        <div class="menu-icon" @click="showAdmin = true">☰</div>
        <div class="logo-wrap"><span class="logo-text">Sabor Prohibido</span></div>
        <div style="display:flex;gap:8px;align-items:center;">
          <button class="history-btn" @click="showHistory = true">
            📋 <span class="history-badge">{{ savedOrders.length }}</span>
          </button>
          <button class="cart-btn" @click="showCart = true">
            🛒 <span>Pedido</span>
            <span class="cart-badge">{{ cartCount() }}</span>
          </button>
        </div>
      </div>
    </header>

    <!-- SEARCH -->
    <div class="search-section">
      <div class="search-inner">
        <span class="search-icon">🔍</span>
        <input v-model="search" type="text" placeholder="Buscar en el menú...">
      </div>
    </div>

    <!-- HERO -->
    <section class="hero">
      <div class="hero-stripe top"></div>
      <h1>Sabor<br>Prohibido</h1>
      <p class="hero-sub">🔥 Sabor Urbano · Entrega Rápida · San Gil</p>
      <div class="promo-tags">
        <span class="promo-tag">🔥😏🍑 Sabor que provoca</span>
        <span class="promo-tag hot">😈🍆💦 Puro fuego</span>
        <span class="promo-tag spicy">🥵💋🔥 No te vas a resistir</span>
      </div>
      <div class="hero-stripe bot"></div>
    </section>

    <!-- CATEGORÍAS -->
    <div class="categories-wrap">
      <div class="cat-scroll">
        <div
          v-for="cat in categories"
          :key="cat.id"
          :class="['cat-chip', { active: selectedCat === cat.id }]"
          @click="selectedCat = cat.id"
        >
          {{ cat.icon }} {{ cat.name }}
        </div>
      </div>
    </div>

    <!-- PRODUCTOS -->
    <div class="products-wrap">
      <div v-if="getGroupedProducts().length === 0" class="no-results">
        <div style="font-size:3rem">😕</div>
        <p>No encontramos ese producto</p>
      </div>

      <div v-for="group in getGroupedProducts()" :key="group.catId">
        <div class="sec-head">
          <span class="sec-title">{{ group.catIcon }} {{ group.catName }}</span>
          <div class="stripe-line"></div>
          <span class="count-pill">{{ group.items.length }}</span>
        </div>
        <div class="product-grid">
          <div
            v-for="(p, i) in group.items"
            :key="p.id"
            class="product-card"
            :style="{ animationDelay: (i % 8) * 0.06 + 's' }"
            @click="openDetail(p)"
          >
            <div class="card-img-wrap">
              <img
                :src="getImg(p.img)"
                :alt="p.name"
                class="card-img"
                @error="onImgError($event, p.emoji)"
              >
              <span v-if="p.popular" class="popular-badge">🔥 POPULAR</span>
            </div>
            <div class="card-body">
              <div class="card-name">{{ p.name }}</div>
              <p class="card-desc">{{ p.desc }}</p>
              <div class="card-footer" @click.stop>
                <span class="price">$ {{ formatPrice(p.price) }}</span>
                <div v-if="getQty(p.id) > 0" class="qty-control">
                  <button class="qty-btn" @click="decrease(p.id)">−</button>
                  <span class="qty-num">{{ getQty(p.id) }}</span>
                  <button class="qty-btn" @click="increase(p)">+</button>
                </div>
                <button v-else class="add-btn" @click="increase(p)">+</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- MODAL DETALLE -->
    <transition name="fade">
      <div v-if="detailProduct" class="modal-overlay" @click.self="detailProduct = null">
        <div class="detail-modal">
          <button class="detail-close" @click="detailProduct = null">X</button>
          <div class="detail-img-wrap">
            <img
              :src="getImg(detailProduct.img)"
              :alt="detailProduct.name"
              class="detail-img"
              @error="onImgError($event, detailProduct.emoji)"
            >
          </div>
          <div class="detail-body">
            <div class="detail-name">{{ detailProduct.name }}</div>
            <p class="detail-desc">{{ detailProduct.fullDesc }}</p>
            <div class="detail-footer">
              <span class="detail-price">$ {{ formatPrice(detailProduct.price) }}</span>
              <div v-if="getQty(detailProduct.id) > 0" class="qty-control">
                <button class="qty-btn" style="width:36px;height:36px;font-size:1.2rem" @click="decrease(detailProduct.id)">−</button>
                <span class="qty-num" style="font-size:1.2rem">{{ getQty(detailProduct.id) }}</span>
                <button class="qty-btn" style="width:36px;height:36px;font-size:1.2rem" @click="increase(detailProduct)">+</button>
              </div>
              <button v-else class="detail-add" @click="increase(detailProduct); detailProduct = null">
                AGREGAR AL PEDIDO
              </button>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- CARRITO SIDEBAR -->
    <transition name="fade">
      <div v-if="showCart" class="cart-overlay-bg" @click.self="showCart = false"></div>
    </transition>
    <transition name="slide">
      <div v-if="showCart" class="cart-sidebar">
        <div class="cart-head">
          <h2>🛒 Mi Pedido</h2>
          <button class="close-x" @click="showCart = false">✕</button>
        </div>
        <div class="cart-items">
          <div v-if="!cart.length" class="cart-empty">
            <div class="ee">🍽️</div>
            <p>CARRITO VACÍO</p>
            <p style="font-size:.78rem;margin-top:8px;font-family:Nunito;color:#444;font-weight:700">¡Agrega algo delicioso!</p>
          </div>
          <div v-for="item in cart" :key="item.id" class="cart-item">
            <span class="ci-emoji">{{ item.emoji }}</span>
            <div class="ci-info">
              <div class="ci-name">{{ item.name }}</div>
              <div class="ci-bottom">
                <div class="qty-control">
                  <button class="qty-btn" style="width:26px;height:26px" @click="decrease(item.id)">−</button>
                  <span class="qty-num" style="font-size:.9rem">{{ item.qty }}</span>
                  <button class="qty-btn" style="width:26px;height:26px" @click="increase(item)">+</button>
                </div>
                <span class="ci-total">$ {{ formatPrice(item.price * item.qty) }}</span>
              </div>
            </div>
            <button class="ci-del" @click="removeItem(item.id)">🗑️</button>
          </div>
        </div>
        <div class="cart-summary">
          <div class="sum-row">
            <span>Subtotal ({{ cartCount() }} items)</span>
            <span>$ {{ formatPrice(cartTotal()) }}</span>
          </div>
          <div class="sum-row"><span>Domicilio</span><span style="color:var(--success)">GRATIS</span></div>
          <div class="sum-row total">
            <span>TOTAL</span>
            <span>$ {{ formatPrice(cartTotal()) }}</span>
          </div>
          <button class="checkout-btn" :disabled="!cart.length" @click="goCheckout">CONFIRMAR PEDIDO →</button>
        </div>
      </div>
    </transition>

    <!-- CHECKOUT MODAL -->
    <transition name="fade">
      <div v-if="showCheckout" class="modal-overlay" @click.self="showCheckout = false">
        <div class="checkout-modal">

          <!-- PASO 1: FORMULARIO -->
          <template v-if="step === 1">
            <div class="form-head">
              <span class="form-title">📋 TU PEDIDO</span>
              <button class="close-x" @click="showCheckout = false">✕</button>
            </div>
            <div class="form-body">
              <div class="form-row">
                <div class="form-group">
                  <label>Nombre *</label>
                  <input v-model.trim="form.nombre" :class="{ error: err.nombre }" placeholder="Tu nombre" @input="clearErr('nombre')">
                  <div v-if="err.nombre" class="error-msg">⚠ {{ err.nombre }}</div>
                </div>
                <div class="form-group">
                  <label>Apellido *</label>
                  <input v-model.trim="form.apellido" :class="{ error: err.apellido }" placeholder="Tu apellido" @input="clearErr('apellido')">
                  <div v-if="err.apellido" class="error-msg">⚠ {{ err.apellido }}</div>
                </div>
              </div>
              <div class="form-group">
                <label>Teléfono / WhatsApp *</label>
                <input v-model.trim="form.tel" :class="{ error: err.tel }" placeholder="3001234567" maxlength="10" @input="clearErr('tel')">
                <div v-if="err.tel" class="error-msg">⚠ {{ err.tel }}</div>
              </div>
              <div class="form-group">
                <label>Tipo de entrega</label>
                <select v-model="form.tipo">
                  <option value="domicilio">🛵 A domicilio</option>
                  <option value="local">🏪 Recoger en local</option>
                </select>
              </div>
              <div v-if="form.tipo === 'domicilio'" class="form-group">
                <label>Dirección *</label>
                <input v-model.trim="form.dir" :class="{ error: err.dir }" placeholder="Calle, barrio..." @input="clearErr('dir')">
                <div v-if="err.dir" class="error-msg">⚠ {{ err.dir }}</div>
              </div>
              <div class="form-group">
                <label>Método de pago *</label>
                <select v-model="form.pago">
                  <option value="">Seleccionar...</option>
                  <option value="efectivo">💵 Efectivo</option>
                  <option value="nequi">📱 Nequi</option>
                  <option value="daviplata">💙 Daviplata</option>
                  <option value="tarjeta">💳 Tarjeta</option>
                </select>
                <div v-if="err.pago" class="error-msg">⚠ {{ err.pago }}</div>
              </div>
              <div v-if="form.pago === 'efectivo'" class="form-group">
                <label>¿Con cuánto paga?</label>
                <input v-model="form.cambio" type="number" placeholder="Ej: 50000">
              </div>
              <div class="form-group">
                <label>Notas (opcional)</label>
                <textarea v-model="form.nota" rows="2" placeholder="Sin cebolla, extra salsa..."></textarea>
              </div>
            </div>
            <div class="form-actions">
              <button class="btn-back" @click="showCheckout = false">Cancelar</button>
              <button class="btn-confirm" @click="validate">VER FACTURA →</button>
            </div>
          </template>

          <!-- PASO 2: FACTURA -->
          <template v-if="step === 2">
            <div class="inv-head-bar">
              <span class="form-title">🧾 FACTURA</span>
              <button class="close-x" @click="showCheckout = false">✕</button>
            </div>
            <div class="invoice-wrap">
              <div class="invoice" id="invoice-printable">
                <div class="inv-top">
                  <div class="inv-logo-text">Sabor Prohibido</div>
                  <div class="inv-sub">NIT: 900.123.456-7 · San Gil, Santander · 📞 3001234567</div>
                </div>
                <div class="inv-meta">
                  <div class="inv-cell"><label>Factura #</label><span>{{ invNum }}</span></div>
                  <div class="inv-cell"><label>Fecha</label><span>{{ invDate }}</span></div>
                  <div class="inv-cell"><label>Cliente</label><span>{{ form.nombre }} {{ form.apellido }}</span></div>
                  <div class="inv-cell"><label>Teléfono</label><span>{{ form.tel }}</span></div>
                  <div class="inv-cell"><label>Entrega</label><span>{{ form.tipo === 'domicilio' ? 'Domicilio' : 'Local' }}</span></div>
                  <div class="inv-cell"><label>Pago</label><span>{{ getPagoLabel() }}</span></div>
                  <div v-if="form.tipo === 'domicilio'" class="inv-cell wide"><label>Dirección</label><span>{{ form.dir }}</span></div>
                  <div v-if="form.nota" class="inv-cell wide"><label>Notas</label><span>{{ form.nota }}</span></div>
                </div>
                <div class="inv-table-wrap">
                  <table>
                    <thead>
                      <tr><th>Producto</th><th>Cant</th><th>Precio</th><th>Total</th></tr>
                    </thead>
                    <tbody>
                      <tr v-for="it in cart" :key="it.id">
                        <td>{{ it.emoji }} {{ it.name }}</td>
                        <td>{{ it.qty }}</td>
                        <td>$ {{ formatPrice(it.price) }}</td>
                        <td>$ {{ formatPrice(it.price * it.qty) }}</td>
                      </tr>
                    </tbody>
                  </table>
                </div>
                <div class="inv-totals">
                  <div class="inv-tot"><span>Subtotal</span><span>$ {{ formatPrice(cartTotal()) }}</span></div>
                  <div class="inv-tot"><span>Domicilio</span><span>GRATIS</span></div>
                  <div class="inv-tot"><span>IVA</span><span>$ 0</span></div>
                  <div class="inv-tot grand"><span>TOTAL A PAGAR</span><span>$ {{ formatPrice(cartTotal()) }}</span></div>
                  <div v-if="form.pago === 'efectivo' && form.cambio" class="inv-tot" style="color:#006600;margin-top:6px">
                    <span>Cambio estimado</span>
                    <span>$ {{ formatPrice(Math.max(0, form.cambio - cartTotal())) }}</span>
                  </div>
                </div>
                <div class="inv-foot-note">GRACIAS POR TU PEDIDO! 🔥😏 SABOR PROHIBIDO</div>
              </div>
            </div>
            <div class="form-actions">
              <button class="btn-back" @click="step = 1">← EDITAR</button>
              <button class="btn-download" @click="downloadPDF">⬇ DESCARGAR PDF</button>
              <button class="btn-confirm" @click="doOrder">🚀 ¡HACER PEDIDO!</button>
            </div>
          </template>

          <!-- PASO 3: ÉXITO -->
          <template v-if="step === 3">
            <div class="success-box">
              <div class="success-big">🎉</div>
              <h2>¡Confirmado!</h2>
              <p>Tu pedido está siendo preparado</p>
              <div class="order-num">ORDEN #{{ invNum }}</div>
              <p style="margin-bottom:24px;color:var(--yellow);font-weight:800;">
                {{ form.tipo === 'domicilio' ? '🛵 Tiempo estimado: 25–40 min' : '🏪 Listo en: 15–20 min' }}
              </p>
              <button class="btn-download" style="margin-bottom:12px;width:100%" @click="downloadPDF">⬇ DESCARGAR FACTURA PDF</button>
              <button class="checkout-btn" style="margin:0" @click="resetAll">VOLVER AL MENÚ</button>
            </div>
          </template>

        </div>
      </div>
    </transition>

    <!-- ══════════════ HISTORIAL DE PEDIDOS ══════════════ -->
    <transition name="fade">
      <div v-if="showHistory" class="modal-overlay" @click.self="showHistory = false">
        <div class="history-modal">
          <div class="form-head">
            <span class="form-title">📋 HISTORIAL DE PEDIDOS</span>
            <button class="close-x" @click="showHistory = false">✕</button>
          </div>
          <div class="history-body">
            <div v-if="!savedOrders.length" style="text-align:center;padding:48px 20px">
              <div style="font-size:3.5rem;margin-bottom:12px">📭</div>
              <p style="font-family:Anton,cursive;font-size:1.2rem;letter-spacing:2px;color:#444">SIN PEDIDOS AÚN</p>
            </div>
            <div v-for="order in savedOrders.slice().reverse()" :key="order.invNum" class="history-item">
              <div class="history-item-head">
                <div>
                  <span class="history-num">{{ order.invNum }}</span>
                  <span class="history-date">{{ order.invDate }}</span>
                </div>
                <div style="display:flex;gap:8px;align-items:center">
                  <span class="history-total">$ {{ formatPrice(order.total) }}</span>
                  <button class="btn-history-pdf" @click="downloadSavedPDF(order)">⬇ PDF</button>
                  <button class="ci-del" @click="deleteOrder(order.invNum)">🗑️</button>
                </div>
              </div>
              <div class="history-client">{{ order.nombre }} {{ order.apellido }} · {{ order.tel }}</div>
              <div class="history-products">
                <span v-for="it in order.items" :key="it.id" class="h-prod-chip">
                  {{ it.emoji }} {{ it.name }} x{{ it.qty }}
                </span>
              </div>
              <div style="display:flex;gap:8px;margin-top:6px">
                <span class="h-tag">{{ order.tipo === 'domicilio' ? '🛵 Domicilio' : '🏪 Local' }}</span>
                <span class="h-tag">{{ getPagoLabelStatic(order.pago) }}</span>
              </div>
            </div>
          </div>
          <div v-if="savedOrders.length" style="padding:0 20px 20px;display:flex;gap:10px">
            <button class="btn-back" style="flex:1" @click="clearHistory">🗑️ Borrar historial</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- ══════════════ PANEL ADMIN – AGREGAR PRODUCTO ══════════════ -->
    <transition name="fade">
      <div v-if="showAdmin" class="modal-overlay" @click.self="showAdmin = false">
        <div class="checkout-modal" style="max-width:520px">
          <div class="form-head">
            <span class="form-title">➕ AGREGAR PRODUCTO</span>
            <button class="close-x" @click="showAdmin = false">✕</button>
          </div>
          <div class="form-body">
            <div class="form-row">
              <div class="form-group">
                <label>Nombre del producto *</label>
                <input v-model.trim="newP.name" :class="{ error: newPErr.name }" placeholder="Ej: Burger Especial" @input="newPErr.name=''">
                <div v-if="newPErr.name" class="error-msg">⚠ {{ newPErr.name }}</div>
              </div>
              <div class="form-group">
                <label>Emoji *</label>
                <input v-model.trim="newP.emoji" :class="{ error: newPErr.emoji }" placeholder="🍔" maxlength="4" @input="newPErr.emoji=''">
                <div v-if="newPErr.emoji" class="error-msg">⚠ {{ newPErr.emoji }}</div>
              </div>
            </div>
            <div class="form-row">
              <div class="form-group">
                <label>Precio (COP) *</label>
                <input v-model.number="newP.price" :class="{ error: newPErr.price }" type="number" placeholder="21900" @input="newPErr.price=''">
                <div v-if="newPErr.price" class="error-msg">⚠ {{ newPErr.price }}</div>
              </div>
              <div class="form-group">
                <label>Categoría *</label>
                <select v-model="newP.cat" :class="{ error: newPErr.cat }" @change="newPErr.cat=''">
                  <option value="">Seleccionar...</option>
                  <option v-for="c in categories.filter(c=>c.id!=='all')" :key="c.id" :value="c.id">{{ c.icon }} {{ c.name }}</option>
                </select>
                <div v-if="newPErr.cat" class="error-msg">⚠ {{ newPErr.cat }}</div>
              </div>
            </div>
            <div class="form-group">
              <label>Descripción corta *</label>
              <input v-model.trim="newP.desc" :class="{ error: newPErr.desc }" placeholder="160g carne, queso, salsas..." @input="newPErr.desc=''">
              <div v-if="newPErr.desc" class="error-msg">⚠ {{ newPErr.desc }}</div>
            </div>
            <div class="form-group">
              <label>Descripción completa</label>
              <textarea v-model="newP.fullDesc" rows="3" placeholder="Descripción detallada para el modal..."></textarea>
            </div>
            <div class="form-group">
              <label>URL de imagen (opcional)</label>
              <input v-model.trim="newP.img" placeholder="https://...">
            </div>
            <div class="form-group" style="display:flex;align-items:center;gap:12px">
              <input type="checkbox" v-model="newP.popular" id="popular-check" style="width:18px;height:18px;accent-color:#FF6B00;cursor:pointer">
              <label for="popular-check" style="margin:0;cursor:pointer;font-size:.85rem;color:#aaa">Marcar como 🔥 POPULAR</label>
            </div>
            <div v-if="newP.name || newP.emoji || newP.price" class="product-preview">
              <div style="font-size:.7rem;color:#555;font-family:Anton,cursive;letter-spacing:1px;margin-bottom:8px">VISTA PREVIA</div>
              <div style="display:flex;align-items:center;gap:12px;background:#1a1a1a;border-radius:8px;padding:10px 14px">
                <span style="font-size:2rem">{{ newP.emoji || '❓' }}</span>
                <div>
                  <div style="font-family:Anton,cursive;color:#FF6B00;font-size:.95rem;letter-spacing:1px">{{ newP.name || 'Nombre del producto' }}</div>
                  <div style="color:#888;font-size:.75rem">{{ newP.desc || 'Descripción...' }}</div>
                  <div style="font-family:Anton,cursive;color:#FFE600;font-size:1.1rem">$ {{ formatPrice(newP.price || 0) }}</div>
                </div>
              </div>
            </div>
          </div>
          <div class="form-actions">
            <button class="btn-back" @click="showAdmin = false">Cancelar</button>
            <button class="btn-confirm" @click="addProduct">✅ AGREGAR PRODUCTO</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- FOOTER -->
    <footer>
      <div class="footer-logo">Sabor Prohibido</div>
      <div class="footer-info">📞 3001234567 · San Gil, Santander · Lun–Dom 10am–11pm</div>
    </footer>

  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'

// ──────────────── JSPDF – carga dinámica desde CDN ────────────────
// No necesitas instalar nada. jsPDF se carga automáticamente al montar.
let jsPDFLib = null
onMounted(async () => {
  // Cargar jsPDF desde CDN
  if (!window.jspdf) {
    const script = document.createElement('script')
    script.src = 'https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js'
    document.head.appendChild(script)
    await new Promise(r => { script.onload = r })
  }
  jsPDFLib = window.jspdf.jsPDF

  // Cargar pedidos guardados
  loadOrders()
})

// ──────────────── ESTADO ────────────────
const search        = ref('')
const selectedCat   = ref('all')
const showCart      = ref(false)
const showCheckout  = ref(false)
const showHistory   = ref(false)
const showAdmin     = ref(false)
const step          = ref(1)
const loading       = ref(false)
const loadingText   = ref('PROCESANDO...')
const notifs        = ref([])
const cart          = ref([])
const detailProduct = ref(null)
const invNum        = ref('')
const invDate       = ref('')
const savedOrders   = ref([])

// Snapshot del pedido confirmado para PDF post-confirmación
const confirmedSnapshot = reactive({ items: [], form: {}, invNum: '', invDate: '', total: 0 })

const form = reactive({
  nombre: '', apellido: '', tel: '',
  tipo: 'domicilio', dir: '',
  pago: '', cambio: '', nota: ''
})
const err = reactive({})

// Nuevo producto
const newP = reactive({ name: '', emoji: '', price: '', cat: '', desc: '', fullDesc: '', img: '', popular: false })
const newPErr = reactive({})

// ──────────────── DATOS ────────────────
const categories = [
  { id: 'all',          name: 'Todo',            icon: '🍽️' },
  { id: 'hamburguesas', name: 'Hamburguesas',     icon: '🍔' },
  { id: 'perros',       name: 'Perros Calientes', icon: '🌭' },
  { id: 'pizzas',       name: 'Pizzas',           icon: '🍕' },
  { id: 'tacos',        name: 'Tacos & Wraps',    icon: '🌮' },
  { id: 'pollo',        name: 'Pollo',            icon: '🍗' },
  { id: 'acompa',       name: 'Acompañamientos',  icon: '🍟' },
  { id: 'bebidas',      name: 'Bebidas',           icon: '🥤' },
  { id: 'postres',      name: 'Postres',           icon: '🍨' },
]

const products = ref([
  { id:1,  name:'Calle Bronx',            desc:'160g carne res a la parrilla, mozzarella, cheddar',           fullDesc:'Mezcla callejera con 160g de deliciosa y jugosa carne de res a la parrilla, queso mozzarella, queso cheddar, tocineta ahumada, cebolla caramelizada, vegetales frescos, salsas de la casa.',                  price:21900, emoji:'🍔', img:'https://media.istockphoto.com/id/1203038816/es/foto/21-hamburguesasobres-con-fondo-negro-para-el-men%C3%BA-hamburguesas-blancas-y-negras-con-carne.jpg?s=170667a&w=0&k=20&c=B_nXv_9PrXFbbO1Vv904ehQp8I23LXv9rglAWxwRgX0=',      cat:'hamburguesas', popular:true  },
  { id:2,  name:'Calle Picasso',          desc:'Pollo crispy, queso gouda, salsa BBQ, vegetales',             fullDesc:'Jugosa pechuga de pollo crispy apanada, queso gouda fundido, tocineta, vegetales frescos, lechuga, tomate y nuestra salsa BBQ especial.',                                                                          price:21900, emoji:'🍔', img:'https://img.freepik.com/fotos-premium/hamburguesa-sobre-fondo-negro-menu_127425-580.jpg',    cat:'hamburguesas', popular:true  },
  { id:3,  name:'Calle Campesina',        desc:'Doble carne angus, queso cheddar doble, tocino',              fullDesc:'Doble carne angus 200g, queso cheddar doble derretido, tocino ahumado crujiente, lechuga, tomate, cebolla morada y salsa campesina exclusiva.',                                                                             price:23900, emoji:'🍔', img:'https://img.pikbest.com/photo/20240708/burger-image-with-black-background-_10658058.jpg!w700wp',  cat:'hamburguesas'                },
  { id:4,  name:'Calle Parlache',         desc:'Triple carne, onion ring, queso cheddar+mozza',               fullDesc:'La reina: triple carne 280g, cheddar y mozzarella fundidos, onion ring crocante, tocino doble, jalapeños, vegetales y salsas secretas de Callejeros.',                                                                                               price:24900, emoji:'🍔', img:'https://www.shutterstock.com/image-photo/freshly-made-cheeseburger-showcases-juicy-600nw-2655640343.jpg',   cat:'hamburguesas'                },
  { id:5,  name:'Calle Graffiti',         desc:'Carne, guacamole fresco, pico de gallo, jalapeños',           fullDesc:'Carne a la parrilla 140g con toque mexicano: guacamole fresco, pico de gallo casero, jalapeños, queso oaxaca fundido y crema agria.',                                                                              price:22900, emoji:'🍔', img:'https://img.pikbest.com/backgrounds/20250227/delicious-cheeseburger-with-bacon-on-dark-background_11559846.jpg!f305cw',   cat:'hamburguesas'                },
  { id:6,  name:'Chicken Callejero',      desc:'Pechuga apanada estilo sureño, coleslaw',                     fullDesc:'Pechuga de pollo apanada estilo sureño, ensalada coleslaw casera, pepinillos encurtidos, queso cheddar y mayonesa de ajo.',                                                                                     price:19900, emoji:'🍔', img:'https://static.vecteezy.com/system/resources/previews/026/971/260/large_2x/hamburger-on-a-black-background-close-up-isolate-free-photo.jpg',    cat:'hamburguesas'                },
  { id:7,  name:'Perro Callejero Clásico',desc:'Salchicha, papas chips, queso, salsas callejeras',            fullDesc:'La estrella del local. Salchicha jumbo con papas chips, queso rallado, piña, salsas callejeras: rosada, mostaza, kétchup y mayonesa.',                                                                                                              price:9900,  emoji:'🌭', img:'https://us.123rf.com/450wm/makstorm/makstorm2305/makstorm230500248/205328111-perro-caliente-jugoso-con-especias-aderezos-ketchup-mayonesa-y-ensalada-fresca-colorido-y.jpg?ver=6',     cat:'perros',       popular:true  },
  { id:8,  name:'Perro Relleno de Queso', desc:'Salchicha rellena mozarella, tocino crujiente',               fullDesc:'Salchicha rellena de queso mozarella fundido, envuelta en tocino crujiente, con papas chips, salsas especiales y queso parmesano rallado.',                                                                                                         price:12900, emoji:'🌭', img:'https://img.freepik.com/fotos-premium/delicioso-perrito-caliente-aislado-sobre-fondo-negro-ai-generado-comida-rapida-comida-callejera_59529-2530.jpg',     cat:'perros'                      },
  { id:9,  name:'Mega Perro XXL',         desc:'Salchicha 25cm, maíz, pimentón, 5 salsas',                    fullDesc:'Salchicha XXL de 25cm con papas fritas, maíz tierno, pimentón rojo, cebolla caramelizada, queso fundido y las 5 salsas de la casa.',                                                                                                                price:15900, emoji:'🌭', img:'https://us.123rf.com/450wm/virtosmedia/virtosmedia2301/virtosmedia230119447/197671015-hot-dog-with-flames-isolated-on-black-background-3d-rendering.jpg',         cat:'perros'                      },
  { id:10, name:'Perro BBQ Smoky',        desc:'Salchicha ahumada, BBQ artesanal, cebolla',                   fullDesc:'Salchicha ahumada especial a la plancha, bañada en salsa BBQ artesanal, cebolla caramelizada, queso gouda y papas tipo rústico.',                                                                                                                   price:13900, emoji:'🌭', img:'https://thumbs.dreamstime.com/b/perro-caliente-con-salsa-sobre-fondo-negro-169495000.jpg',         cat:'perros'                      },
  { id:11, name:'Pizza Margherita',       desc:'Tomate, mozzarella fresca, albahaca',                         fullDesc:'Base de tomate artesanal, mozzarella fresca abundante, albahaca fresca y aceite de oliva virgen.',                                                                                                                                                  price:22900, emoji:'🍕', img:'https://img.freepik.com/foto-gratis/deliciosa-pizza-estudio_23-2151846547.jpg',  cat:'pizzas'                      },
  { id:12, name:'Pizza BBQ Pollo',        desc:'Pollo desmenuzado, pimentón, salsa BBQ',                      fullDesc:'Pollo desmenuzado marinado en BBQ, pimentón rojo y verde, cebolla morada, queso mozzarella doble y salsa BBQ callejera.',                                                                                                                   price:25900, emoji:'🍕', img:'https://st4.depositphotos.com/3316741/20026/i/450/depositphotos_200263422-stock-photo-pepperoni-pizza-on-black-concrete.jpg',         cat:'pizzas',       popular:true  },
  { id:13, name:'Pizza 4 Quesos',         desc:'Mozzarella, cheddar, parmesano, brie fundido',                fullDesc:'La extravagancia hecha pizza: mozzarella, cheddar, parmesano y brie sobre base de tomate y aceite de trufa.',                                                                                                                                      price:27900, emoji:'🍕', img:'https://static.vecteezy.com/system/resources/previews/003/414/241/large_2x/top-view-of-pizza-cheese-ham-bacon-pepperoni-isolated-black-background-free-photo.jpg',     cat:'pizzas'                      },
  { id:14, name:'Pizza Pepperoni',        desc:'Pepperoni premium, queso doble, tomate',                      fullDesc:'Generosa capa de pepperoni premium, doble mozzarella, salsa de tomate callejera con ajo y hierbas.',                                                                                                                                                price:24900, emoji:'🍕', img:'https://thumbs.dreamstime.com/b/fondo-negro-de-pizza-para-publicidad-pizzas-negras-en-la-mesa-190272478.jpg',   cat:'pizzas'                      },
  { id:15, name:'Taco Carne Asada',       desc:'3 tacos carne marinada, cilantro, cebolla morada',            fullDesc:'3 tacos de maíz con carne de res marinada en adobo, cilantro fresco, cebolla morada, jalapeños, crema agria y salsa verde.',                                                                                                                       price:13900, emoji:'🌮', img:'https://thumbs.dreamstime.com/b/salsa-tacos-fondo-negro-361017496.jpg',        cat:'tacos',        popular:true  },
  { id:16, name:'Taco Pollo Picante',     desc:'3 tacos pollo chipotle, guacamole, ají',                      fullDesc:'3 tacos con pollo jugoso al chipotle, guacamole fresco, pico de gallo, ají, crema agria y queso cotija.',                                                                                                                                          price:14900, emoji:'🌮', img:'https://thumbs.dreamstime.com/b/delicioso-taco-flotando-con-ingredientes-ca%C3%ADdos-sobre-fondo-negro-flotante-relleno-de-cebolla-tomate-y-cilantro-en-un-generado-397720488.jpg',        cat:'tacos'                       },
  { id:17, name:'Wrap Fitness Pollo',     desc:'Tortilla integral, pollo plancha, yogur limón',               fullDesc:'Tortilla de trigo integral, pechuga de pollo a la plancha, lechuga, tomate, aguacate, zanahoria rallada y aderezo de yogur y limón.',                                                                                                              price:13900, emoji:'🫓', img:'https://static.vecteezy.com/system/resources/previews/028/630/157/non_2x/tacos-with-savory-meat-and-colorful-vegetables-authentic-mexican-cuisine-on-a-black-background-ai-generated-free-photo.JPG',      cat:'tacos'                       },
  { id:18, name:'Taco Vegano',            desc:'3 tacos frijoles negros, aguacate, salsa verde',              fullDesc:'3 tacos 100% vegetales: frijoles negros especiados, aguacate, col morada, tomate cherry, maíz y salsa verde tomatillo.',                                                                                                                          price:12900, emoji:'🌮', img:'https://img.freepik.com/foto-gratis/envolturas-burritos-carne-res-vegetales-imagen-generada-ia_511042-1674.jpg?semt=ais_incoming&w=740&q=80',       cat:'tacos'                       },
  { id:19, name:'Pechuga Crispy + Papas', desc:'Pechuga apanada, papas, coleslaw casero',                    fullDesc:'Gran pechuga de pollo apanada con especias callejeras, papas fritas a la francesa crujientes y ensalada coleslaw casera con mayonesa de limón.',                                                                                                    price:16900, emoji:'🍗', img:'https://static.vecteezy.com/system/resources/previews/027/671/368/large_2x/crispy-fried-chicken-on-the-wooden-board-with-dark-lighting-and-black-background-food-and-delivery-concept-generative-ai-free-photo.jpg',      cat:'pollo',        popular:true  },
  { id:20, name:'Alitas BBQ x6',          desc:'6 alitas bañadas en BBQ ahumada artesanal',                  fullDesc:'6 alitas de pollo horneadas y luego fritas, bañadas en salsa BBQ ahumada artesanal. Con dip de crema agria y bastones de apio.',                                                                                                                   price:18900, emoji:'🍗', img:'https://png.pngtree.com/thumb_back/fh260/background/20250503/pngtree-chicken-wings-with-hot-sauce-splashing-around-against-a-black-background-image_17256019.jpg',        cat:'pollo'                       },
  { id:21, name:'Alitas Búfalo x6',       desc:'6 alitas picantes, dip queso azul',                          fullDesc:'6 alitas crujientes en salsa búfalo picante con mantequilla, vinagre y ají. Con dip de queso azul.',                                                                                                                                               price:19900, emoji:'🍗', img:'https://png.pngtree.com/background/20250522/original/pngtree-chicken-wings-with-hot-sauce-splashing-around-against-a-black-background-picture-image_16509347.jpg',     cat:'pollo'                       },
  { id:22, name:'Combo Familiar Pollo',   desc:'Pollo entero, papas familiares, 4 bebidas',                  fullDesc:'Pollo entero asado con especias callejeras, papas fritas en porción familiar, ensalada coleslaw grande y 4 gaseosas personales.',                                                                                                                   price:49900, emoji:'🍗', img:'https://img.freepik.com/fotos-premium/pollo-frito-patatas-fritas_1410957-66500.jpg',       cat:'pollo',        popular:true  },
  { id:23, name:'Papas Fritas Medianas',  desc:'Papas crujientes, sal y salsa a elegir',                     fullDesc:'Papas a la francesa cortadas a mano, fritas en aceite vegetal. Elige tu salsa: BBQ, rosada, mayonesa o kétchup.',                                                                                                                                  price:5900,  emoji:'🍟', img:'https://img.freepik.com/foto-gratis/deliciosas-papas-fritas-estudio_23-2151846534.jpg?semt=ais_hybrid&w=740&q=80',      cat:'acompa'                      },
  { id:24, name:'Papas Callejeras',       desc:'Papas, queso fundido, tocino, cebollín',                     fullDesc:'Papas fritas cubiertas con queso cheddar fundido, tocino crujiente, cebollín picado, crema agria y jalapeños encurtidos.',                                                                                                                          price:8900,  emoji:'🍟', img:'https://img.freepik.com/fotos-premium/monton-papas-fritas-sobre-fondo-negro_135427-8040.jpg',  cat:'acompa',       popular:true  },
  { id:25, name:'Onion Rings x8',         desc:'Aros de cebolla crocantes, dip BBQ',                         fullDesc:'8 aros de cebolla dulce apanados en mezcla de cerveza y especias, fritos hasta quedar dorados y crujientes. Con dip BBQ.',                                                                                                                         price:7900,  emoji:'🧅', img:'https://img.freepik.com/fotos-premium/aros-cebolla-fritos-sobre-fondo-negro-ia-generativa_74760-2033.jpg',       cat:'acompa'                      },
  { id:26, name:'Papa Frita Normal',      desc:'Yuca colombiana dorada, guacamole, ajo',                     fullDesc:'Yuca colombiana cocida y frita hasta quedar dorada y crujiente por fuera, suave por dentro. Con guacamole fresco y salsa de ajo.',                                                                                                                  price:6900,  emoji:'🌿', img:'https://img.freepik.com/fotos-premium/plato-papas-fritas-sobre-fondo-negro_192217-1557.jpg',        cat:'acompa'                      },
  { id:27, name:'Gaseosa Personal 350ml', desc:'Coca-Cola, Pepsi, Sprite o agua',                            fullDesc:'Gaseosa helada a elección: Coca-Cola, Pepsi, Sprite, 7UP o agua mineral Cristal. Con hielo y limón a solicitud.',                                                                                                                                  price:3900,  emoji:'🥤', img:'https://st3.depositphotos.com/9089900/19068/i/450/depositphotos_190689130-stock-photo-kazan-russia-march-17-2018.jpg',    cat:'bebidas'                     },
  { id:28, name:'Jugo Natural',           desc:'Mango, lulo, maracuyá o mora',                               fullDesc:'Jugo natural preparado al momento con frutas frescas. Disponible en: mango, lulo, maracuyá, mora, guanábana o naranja. En agua o leche.',                                                                                                          price:5900,  emoji:'🍹', img:'https://png.pngtree.com/thumb_back/fh260/background/20240528/pngtree-different-juices-on-black-background-image_15823155.jpg',       cat:'bebidas',      popular:true  },
  { id:29, name:'Malteada Oreo',          desc:'Malteada cremosa Oreo, crema chantilly',                     fullDesc:'Malteada espesa con helado de vainilla, leche entera, Oreo triturada y coronada con crema chantilly y más Oreo.',                                                                                                                                  price:8900,  emoji:'🥛', img:'https://images.unsplash.com/photo-1641665271888-575e46923776?fm=jpg&q=60&w=3000',   cat:'bebidas'                     },
  { id:30, name:'Limonada de Coco',       desc:'Limonada helada, crema de coco, leche condensada',           fullDesc:'Limonada de limón tahití con crema de coco, leche condensada y hielo frappé. Refrescante y tropical.',                                                                                                                                             price:6900,  emoji:'🍋', img:'https://image.slidesdocs.com/responsive-images/background/coconut-juice-and-fruit-rendered-in-3d-on-a-black-powerpoint-background_95d5edc235__960_540.jpg',   cat:'bebidas',      popular:true  },
  { id:31, name:'Brownie + Helado',       desc:'Brownie tibio de chocolate, helado vainilla',                fullDesc:'Brownie de chocolate belga tibio, húmedo y fudgy, servido con bola de helado de vainilla, salsa de chocolate caliente y nueces.',                                                                                                                  price:7900,  emoji:'🍫', img:'https://www.shutterstock.com/image-photo/chocolate-cake-topped-ice-cream-600nw-2236290477.jpg',    cat:'postres',      popular:true  },
  { id:32, name:'Churros con Arequipe',   desc:'5 churros crocantes con arequipe casero',                   fullDesc:'5 churros artesanales fritos al momento, rebozados en azúcar y canela, servidos con abundante arequipe casero y salsa de chocolate.',                                                                                                               price:6900,  emoji:'🍩', img:'https://thumbs.dreamstime.com/b/churros-con-el-az-car-sumergi-en-salsa-de-chocolate-un-fondo-negro-palillos-churro-pasteles-fritos-la-pasta-visi-n-superior-155119316.jpg',    cat:'postres'                     },
  { id:33, name:'Waffle Nutella',         desc:'Waffle esponjoso, Nutella, fresas frescas',                  fullDesc:'Waffle esponjoso recién hecho con cobertura de Nutella, fresas frescas laminadas, crema chantilly y azúcar glass.',                                                                                                                                price:9900,  emoji:'🧇', img:'https://www.shutterstock.com/image-photo/maxi-gourmet-waffle-whipped-cream-600w-2560780991.jpg',     cat:'postres'                     },
])

// ──────────────── FUNCIONES GENERALES ────────────────
function formatPrice(n) {
  return Number(n).toLocaleString('es-CO')
}
function getImg(url) { return url }

function onImgError(event, emoji) {
  const parent = event.target.parentElement
  event.target.style.display = 'none'
  const span = document.createElement('span')
  span.style.fontSize = '4.5rem'
  span.textContent = emoji
  parent.appendChild(span)
}

function cartCount() { return cart.value.reduce((s, i) => s + i.qty, 0) }
function cartTotal() { return cart.value.reduce((s, i) => s + i.price * i.qty, 0) }

function getPagoLabel() {
  const labels = { efectivo:'Efectivo', nequi:'Nequi', daviplata:'Daviplata', tarjeta:'Tarjeta' }
  return labels[form.pago] || ''
}
function getPagoLabelStatic(pago) {
  const labels = { efectivo:'💵 Efectivo', nequi:'📱 Nequi', daviplata:'💙 Daviplata', tarjeta:'💳 Tarjeta' }
  return labels[pago] || ''
}

function getGroupedProducts() {
  const q = search.value.toLowerCase().trim()
  const activeCats = selectedCat.value === 'all'
    ? categories.filter(c => c.id !== 'all')
    : categories.filter(c => c.id === selectedCat.value)

  return activeCats.map(cat => {
    let items = products.value.filter(p => p.cat === cat.id)
    if (q) items = items.filter(p =>
      p.name.toLowerCase().includes(q) || p.desc.toLowerCase().includes(q)
    )
    return { catId: cat.id, catName: cat.name, catIcon: cat.icon, items }
  }).filter(g => g.items.length > 0)
}

function getQty(id) {
  const found = cart.value.find(i => i.id === id)
  return found ? found.qty : 0
}

function increase(p) {
  const existing = cart.value.find(i => i.id === p.id)
  if (existing) { existing.qty++ }
  else {
    cart.value.push({ id: p.id, name: p.name, emoji: p.emoji, price: p.price, qty: 1 })
    notify(`${p.emoji} ${p.name} agregado`, 'success')
  }
}

function decrease(id) {
  const item = cart.value.find(i => i.id === id)
  if (!item) return
  if (item.qty > 1) { item.qty-- }
  else { removeItem(id) }
}

function removeItem(id) {
  const idx = cart.value.findIndex(i => i.id === id)
  if (idx !== -1) {
    notify(`${cart.value[idx].name} eliminado`, 'info')
    cart.value.splice(idx, 1)
  }
}

let nId = 0
function notify(text, type = 'info') {
  const icons = { success: '✅', error: '❌', info: 'ℹ️' }
  const id = ++nId
  notifs.value.push({ id, text, type, icon: icons[type] })
  setTimeout(() => {
    const i = notifs.value.findIndex(n => n.id === id)
    if (i !== -1) notifs.value.splice(i, 1)
  }, 3000)
}

function openDetail(p) { detailProduct.value = p }

function goCheckout() {
  if (!cart.value.length) return
  showCart.value = false
  step.value = 1
  showCheckout.value = true
}

function clearErr(field) { delete err[field] }

function validate() {
  Object.keys(err).forEach(k => delete err[k])
  let ok = true
  if (!form.nombre)   { err.nombre  = 'Requerido'; ok = false }
  if (!form.apellido) { err.apellido = 'Requerido'; ok = false }
  if (!form.tel)      { err.tel = 'Requerido'; ok = false }
  else if (!/^3\d{9}$/.test(form.tel)) { err.tel = 'Número inválido (10 dígitos, inicia en 3)'; ok = false }
  if (form.tipo === 'domicilio' && !form.dir) { err.dir = 'Dirección requerida para domicilio'; ok = false }
  if (!form.pago) { err.pago = 'Selecciona método de pago'; ok = false }
  if (!ok) { notify('Corrige los errores del formulario', 'error'); return }

  invNum.value  = 'CAL-' + Date.now().toString().slice(-5)
  const n = new Date()
  invDate.value = n.toLocaleDateString('es-CO', { day: '2-digit', month: 'short', year: 'numeric' })
  step.value = 2
}

function doOrder() {
  // Guardar snapshot antes de limpiar carrito
  Object.assign(confirmedSnapshot, {
    items: JSON.parse(JSON.stringify(cart.value)),
    form: { ...form },
    invNum: invNum.value,
    invDate: invDate.value,
    total: cartTotal()
  })

  showCheckout.value = false
  loading.value = true
  loadingText.value = 'ENVIANDO PEDIDO...'
  setTimeout(() => { loadingText.value = 'CONFIRMANDO CON LA COCINA...' }, 1000)
  setTimeout(() => {
    loading.value = false
    // Guardar pedido
    saveOrder()
    showCheckout.value = true
    step.value = 3
    notify('🎉 ¡Pedido confirmado!', 'success')
  }, 2400)
}

function resetAll() {
  cart.value = []
  Object.assign(form, { nombre: '', apellido: '', tel: '', tipo: 'domicilio', dir: '', pago: '', cambio: '', nota: '' })
  step.value = 1
  showCheckout.value = false
  notify('¡Gracias por tu pedido! Vuelve pronto 🔥', 'success')
}

// ──────────────── PEDIDOS GUARDADOS (localStorage) ────────────────
function saveOrder() {
  const order = {
    invNum: confirmedSnapshot.invNum,
    invDate: confirmedSnapshot.invDate,
    nombre: confirmedSnapshot.form.nombre,
    apellido: confirmedSnapshot.form.apellido,
    tel: confirmedSnapshot.form.tel,
    tipo: confirmedSnapshot.form.tipo,
    dir: confirmedSnapshot.form.dir,
    pago: confirmedSnapshot.form.pago,
    cambio: confirmedSnapshot.form.cambio,
    nota: confirmedSnapshot.form.nota,
    items: confirmedSnapshot.items,
    total: confirmedSnapshot.total
  }
  const all = JSON.parse(localStorage.getItem('burgerflow_orders') || '[]')
  all.push(order)
  localStorage.setItem('saborprohibido_orders', JSON.stringify(all))
  savedOrders.value = all
}

function loadOrders() {
  savedOrders.value = JSON.parse(localStorage.getItem('saborprohibido_orders') || '[]')
}

function deleteOrder(num) {
  const all = savedOrders.value.filter(o => o.invNum !== num)
  localStorage.setItem('burgerflow_orders', JSON.stringify(all))
  savedOrders.value = all
  notify('Pedido eliminado', 'info')
}

function clearHistory() {
  localStorage.removeItem('saborprohibido_orders')
  savedOrders.value = []
  notify('Historial borrado', 'info')
}

// ──────────────── DESCARGAR PDF (jsPDF) ────────────────
// Librería: jsPDF v2.5.1 — se carga automáticamente desde CDN al montar el componente.
// No necesitas instalar nada con npm/yarn.
function buildPDF(orderData) {
  if (!jsPDFLib) { notify('PDF aún cargando, intenta de nuevo', 'error'); return null }

  const doc = new jsPDFLib({ orientation: 'portrait', unit: 'mm', format: 'a5' })
  const W = doc.internal.pageSize.getWidth()
  let y = 0

  // Fondo negro header
  doc.setFillColor(0, 0, 0)
  doc.rect(0, 0, W, 28, 'F')

  // Línea amarilla
  doc.setFillColor(255, 230, 0)
  doc.rect(0, 28, W, 2, 'F')

  // Logo
  doc.setTextColor(255, 230, 0)
  doc.setFontSize(18)
  doc.setFont('helvetica', 'bold')
  doc.text('SABOR PROHIBIDO', W / 2, 13, { align: 'center' })
  doc.setFontSize(7)
  doc.setTextColor(150, 150, 150)
  doc.text('NIT: 900.123.456-7  |  San Gil, Santander  |  Tel: 3001234567', W / 2, 21, { align: 'center' })

  y = 36

  // Info factura
  doc.setFillColor(15, 15, 15)
  doc.rect(0, y - 2, W, 32, 'F')

  const col1 = 10, col2 = W / 2 + 2
  doc.setTextColor(100, 100, 100)
  doc.setFontSize(7)
  doc.setFont('helvetica', 'normal')

  const addField = (lbl, val, cx, cy) => {
    doc.setTextColor(120, 120, 120)
    doc.setFont('helvetica', 'normal')
    doc.setFontSize(6.5)
    doc.text(lbl.toUpperCase(), cx, cy)
    doc.setTextColor(255, 255, 255)
    doc.setFont('helvetica', 'bold')
    doc.setFontSize(8)
    doc.text(String(val), cx, cy + 5)
  }

  addField('Factura #', orderData.invNum, col1, y + 2)
  addField('Fecha', orderData.invDate, col2, y + 2)
  addField('Cliente', `${orderData.nombre} ${orderData.apellido}`, col1, y + 14)
  addField('Telefono', orderData.tel, col2, y + 14)
  addField('Entrega', orderData.tipo === 'domicilio' ? 'A domicilio' : 'Recoger en local', col1, y + 23)
  addField('Pago', getPagoLabelStatic(orderData.pago).replace(/[^\w\s]/gi, ''), col2, y + 23)

  y += 36
  if (orderData.tipo === 'domicilio' && orderData.dir) {
    doc.setFillColor(20, 20, 20)
    doc.rect(0, y - 1, W, 12, 'F')
    addField('Direccion', orderData.dir, col1, y + 2)
    y += 14
  }
  if (orderData.nota) {
    doc.setFillColor(20, 20, 20)
    doc.rect(0, y - 1, W, 12, 'F')
    addField('Notas', orderData.nota, col1, y + 2)
    y += 14
  }

  y += 4
  // Tabla header
  doc.setFillColor(255, 107, 0)
  doc.rect(0, y, W, 9, 'F')
  doc.setTextColor(255, 255, 255)
  doc.setFont('helvetica', 'bold')
  doc.setFontSize(7.5)
  doc.text('PRODUCTO', col1, y + 6)
  doc.text('CANT', W * 0.62, y + 6, { align: 'center' })
  doc.text('PRECIO', W * 0.78, y + 6, { align: 'center' })
  doc.text('TOTAL', W - 8, y + 6, { align: 'right' })
  y += 11

  // Filas
  orderData.items.forEach((it, idx) => {
    if (idx % 2 === 0) {
      doc.setFillColor(20, 20, 20)
      doc.rect(0, y - 1, W, 9, 'F')
    }
    doc.setTextColor(220, 220, 220)
    doc.setFont('helvetica', 'normal')
    doc.setFontSize(7.5)
    doc.text(`${it.name}`, col1, y + 5.5)
    doc.text(String(it.qty), W * 0.62, y + 5.5, { align: 'center' })
    doc.setFont('helvetica', 'normal')
    doc.text(`$${formatPrice(it.price)}`, W * 0.78, y + 5.5, { align: 'center' })
    doc.setFont('helvetica', 'bold')
    doc.setTextColor(255, 230, 0)
    doc.text(`$${formatPrice(it.price * it.qty)}`, W - 8, y + 5.5, { align: 'right' })
    y += 9
  })

  y += 4
  // Totales
  doc.setFillColor(10, 10, 10)
  doc.rect(0, y - 2, W, 30, 'F')
  doc.setFontSize(8)

  const addTotal = (lbl, val, highlight) => {
    doc.setFont('helvetica', 'normal')
    doc.setTextColor(highlight ? 255 : 130, highlight ? 230 : 130, highlight ? 0 : 130)
    doc.text(lbl, col1 + 2, y + 5)
    doc.setFont('helvetica', 'bold')
    doc.setTextColor(highlight ? 255 : 200, highlight ? 230 : 200, highlight ? 0 : 200)
    doc.text(val, W - 8, y + 5, { align: 'right' })
    y += 7
  }

  addTotal('Subtotal', `$${formatPrice(orderData.total)}`, false)
  addTotal('Domicilio', 'GRATIS', false)
  addTotal('IVA', '$0', false)

  // Línea separadora
  doc.setDrawColor(255, 107, 0)
  doc.setLineWidth(0.5)
  doc.line(col1, y, W - col1, y)
  y += 3

  doc.setFontSize(11)
  doc.setFont('helvetica', 'bold')
  doc.setTextColor(255, 230, 0)
  doc.text('TOTAL A PAGAR', col1 + 2, y + 5)
  doc.text(`$${formatPrice(orderData.total)}`, W - 8, y + 5, { align: 'right' })
  y += 12

  if (orderData.pago === 'efectivo' && orderData.cambio) {
    doc.setFontSize(8)
    doc.setFont('helvetica', 'normal')
    doc.setTextColor(0, 200, 100)
    const cambio = Math.max(0, orderData.cambio - orderData.total)
    doc.text(`Cambio estimado: $${formatPrice(cambio)}`, col1 + 2, y + 4)
    y += 10
  }

  // Footer
  const pageH = doc.internal.pageSize.getHeight()
  doc.setFillColor(0, 0, 0)
  doc.rect(0, pageH - 14, W, 14, 'F')
  doc.setFillColor(255, 230, 0)
  doc.rect(0, pageH - 15, W, 1.5, 'F')
  doc.setTextColor(80, 80, 80)
  doc.setFontSize(7)
  doc.setFont('helvetica', 'bold')
  doc.text('GRACIAS POR TU PEDIDO! | SABOR PROHIBIDO | SAN GIL', W / 2, pageH - 6, { align: 'center' })

  return doc
}

function downloadPDF() {
  const orderData = {
    invNum: invNum.value,
    invDate: invDate.value,
    nombre: form.nombre,
    apellido: form.apellido,
    tel: form.tel,
    tipo: form.tipo,
    dir: form.dir,
    pago: form.pago,
    cambio: form.cambio,
    nota: form.nota,
    items: [...cart.value],
    total: cartTotal()
  }
  // Si estamos en paso 3, usar snapshot
  if (step.value === 3 && confirmedSnapshot.items.length) {
    Object.assign(orderData, {
      items: confirmedSnapshot.items,
      total: confirmedSnapshot.total,
      ...confirmedSnapshot.form,
      invNum: confirmedSnapshot.invNum,
      invDate: confirmedSnapshot.invDate,
    })
  }
  const doc = buildPDF(orderData)
  if (doc) {
    doc.save(`Factura-${orderData.invNum}.pdf`)
    notify('📄 PDF descargado', 'success')
  }
}

function downloadSavedPDF(order) {
  const doc = buildPDF(order)
  if (doc) {
    doc.save(`Factura-${order.invNum}.pdf`)
    notify('📄 PDF descargado', 'success')
  }
}

// ──────────────── AGREGAR PRODUCTO ────────────────
function addProduct() {
  Object.keys(newPErr).forEach(k => delete newPErr[k])
  let ok = true
  if (!newP.name)  { newPErr.name  = 'Requerido'; ok = false }
  if (!newP.emoji) { newPErr.emoji = 'Requerido'; ok = false }
  if (!newP.price || newP.price <= 0) { newPErr.price = 'Precio inválido'; ok = false }
  if (!newP.cat)   { newPErr.cat   = 'Selecciona categoría'; ok = false }
  if (!newP.desc)  { newPErr.desc  = 'Requerido'; ok = false }
  if (!ok) { notify('Completa los campos requeridos', 'error'); return }

  const newId = Math.max(...products.value.map(p => p.id)) + 1
  products.value.push({
    id: newId,
    name: newP.name,
    emoji: newP.emoji,
    price: Number(newP.price),
    cat: newP.cat,
    desc: newP.desc,
    fullDesc: newP.fullDesc || newP.desc,
    img: newP.img || '',
    popular: newP.popular
  })

  // Reset form
  Object.assign(newP, { name:'', emoji:'', price:'', cat:'', desc:'', fullDesc:'', img:'', popular:false })
  showAdmin.value = false
  notify(`✅ "${products.value[products.value.length-1].name}" agregado al menú`, 'success')
}
</script>

<style scoped>
/* ══════════════ RESET & BASE ══════════════ */
:root {
  --yellow:  #FFE600;
  --orange:  #FF6B00;
  --red:     #E63946;
  --black:   #000000;
  --dark:    #0a0a0a;
  --card-bg: #111111;
  --text:    #ffffff;
  --muted:   #888888;
  --success: #00e676;
}
*, *::before, *::after { margin:0; padding:0; box-sizing:border-box; }

/* FONDO NEGRO TOTAL – aplica al host y al body */
:host, #app {
  background:#000000 !important;
  min-height:100vh;
  color:#ffffff;
  font-family:'Nunito',sans-serif;
}

/* Asegura que el body también sea negro desde dentro del componente */
body { background:#000000 !important; }

/* ── HEADER ── */
header { background:#000; border-bottom:3px solid #FFE600; position:sticky; top:0; z-index:100; }
.header-inner { max-width:1400px; margin:0 auto; padding:0 20px; height:72px; display:flex; align-items:center; gap:16px; }
.menu-icon { color:#FFE600; font-size:1.6rem; cursor:pointer; flex-shrink:0; }
.logo-wrap { flex:1; display:flex; justify-content:center; }
.logo-text { font-family:'Permanent Marker',cursive; font-size:2.2rem; letter-spacing:2px; background:linear-gradient(180deg,#FFD700 0%,#FF6B00 60%,#FF3300 100%); -webkit-background-clip:text; -webkit-text-fill-color:transparent; background-clip:text; filter:drop-shadow(0 2px 8px rgba(255,180,0,.4)); text-transform:uppercase; }
.cart-btn { background:transparent; border:2px solid #FFE600; border-radius:8px; padding:8px 16px; color:#FFE600; font-family:'Anton',cursive; font-size:.9rem; letter-spacing:1px; cursor:pointer; display:flex; align-items:center; gap:8px; transition:all .2s; flex-shrink:0; }
.cart-btn:hover { background:#FFE600; color:#000; }
.cart-badge { background:#FF6B00; color:#fff; border-radius:50%; width:22px; height:22px; font-size:.75rem; font-weight:900; display:flex; align-items:center; justify-content:center; }

.history-btn { background:transparent; border:2px solid #444; border-radius:8px; padding:8px 14px; color:#888; font-size:.9rem; cursor:pointer; display:flex; align-items:center; gap:6px; transition:all .2s; }
.history-btn:hover { border-color:#FFE600; color:#FFE600; }
.history-badge { background:#333; color:#aaa; border-radius:50%; width:20px; height:20px; font-size:.7rem; font-weight:900; display:flex; align-items:center; justify-content:center; font-family:'Anton',cursive; }

/* ── SEARCH ── */
.search-section { background:#000; padding:14px 20px; border-bottom:1px solid #1a1a1a; }
.search-inner { max-width:1400px; margin:0 auto; position:relative; }
.search-inner input { width:100%; background:#111; border:2px solid #222; border-radius:8px; padding:11px 20px 11px 46px; color:#fff; font-family:'Nunito',sans-serif; font-size:.95rem; outline:none; transition:all .25s; }
.search-inner input:focus { border-color:#FFE600; background:#0d0d0d; }
.search-inner input::placeholder { color:#444; }
.search-icon { position:absolute; left:15px; top:50%; transform:translateY(-50%); font-size:1.1rem; }

/* ── HERO ── */
.hero { background:#000; padding:48px 20px; text-align:center; position:relative; overflow:hidden; }
.hero-stripe { position:absolute; left:0; right:0; height:12px; background:repeating-linear-gradient(-45deg,#FFE600 0px,#FFE600 10px,#000 10px,#000 20px); }
.hero-stripe.top { top:0; }
.hero-stripe.bot { bottom:0; }
.hero h1 { font-family:'Permanent Marker',cursive; font-size:clamp(2.5rem,8vw,5.5rem); background:linear-gradient(180deg,#FFD700 0%,#FF6B00 60%,#FF3300 100%); -webkit-background-clip:text; -webkit-text-fill-color:transparent; background-clip:text; line-height:1; margin-bottom:12px; filter:drop-shadow(0 4px 12px rgba(255,150,0,.5)); }
.hero-sub { font-family:'Anton',cursive; font-size:1.1rem; letter-spacing:3px; color:#FFE600; text-transform:uppercase; }

/* ── CATEGORÍAS ── */
.categories-wrap { background:#000; border-bottom:2px solid #1a1a1a; padding:0 20px; }
.cat-scroll { max-width:1400px; margin:0 auto; display:flex; overflow-x:auto; scrollbar-width:none; }
.cat-scroll::-webkit-scrollbar { display:none; }
.cat-chip { padding:14px 20px; font-family:'Anton',cursive; font-size:.85rem; letter-spacing:1px; color:#555; cursor:pointer; white-space:nowrap; border-bottom:3px solid transparent; transition:all .2s; display:flex; align-items:center; gap:6px; }
.cat-chip:hover { color:#FFE600; }
.cat-chip.active { color:#FFE600; border-bottom-color:#FFE600; }

/* ── PRODUCTOS ── */
.products-wrap { max-width:1400px; margin:0 auto; padding:32px 20px; background:#000; }
.sec-head { display:flex; align-items:center; gap:16px; margin-bottom:24px; }
.sec-title { font-family:'Anton',cursive; font-size:2rem; letter-spacing:3px; color:#FFE600; text-transform:uppercase; white-space:nowrap; }
.stripe-line { flex:1; height:20px; background:repeating-linear-gradient(-45deg,#FFE600 0px,#FFE600 10px,#000 10px,#000 20px); border-radius:2px; max-width:300px; }
.count-pill { background:#FFE600; color:#000; font-family:'Anton',cursive; font-size:.85rem; letter-spacing:1px; padding:4px 14px; border-radius:4px; white-space:nowrap; }
.product-grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(210px,1fr)); gap:16px; margin-bottom:40px; }

/* ── CARD ── */
.product-card { background:#111; border:2px solid #1e1e1e; border-radius:12px; overflow:hidden; cursor:pointer; transition:all .25s; animation:fadeUp .4s ease both; position:relative; }
@keyframes fadeUp { from{opacity:0;transform:translateY(16px)} to{opacity:1;transform:translateY(0)} }
.product-card:hover { transform:translateY(-4px); border-color:#FF6B00; box-shadow:0 0 0 1px #FF6B00,0 12px 40px rgba(255,107,0,.25); }
.card-img-wrap { width:100%; aspect-ratio:1/1; display:flex; align-items:center; justify-content:center; background:#1a1a1a; position:relative; overflow:hidden; }
.card-img { width:100%; height:100%; object-fit:cover; display:block; }
.popular-badge { position:absolute; top:10px; right:10px; background:#FF6B00; color:#fff; font-family:'Anton',cursive; font-size:.7rem; letter-spacing:1px; padding:3px 10px; border-radius:4px; text-transform:uppercase; z-index:1; }
.card-body { padding:12px 14px 14px; }
.card-name { font-family:'Anton',cursive; font-size:1rem; letter-spacing:1.5px; color:#FF6B00; text-transform:uppercase; margin-bottom:4px; line-height:1.2; }
.card-desc { font-size:.78rem; color:#666; line-height:1.4; margin-bottom:12px; min-height:32px; }
.card-footer { display:flex; align-items:center; justify-content:space-between; gap:8px; }
.price { font-family:'Anton',cursive; font-size:1.35rem; letter-spacing:1px; color:#FFE600; }
.add-btn { background:#FFE600; border:none; border-radius:8px; width:36px; height:36px; font-size:1.4rem; font-weight:900; cursor:pointer; display:flex; align-items:center; justify-content:center; color:#000; transition:all .2s; flex-shrink:0; line-height:1; }
.add-btn:hover { background:#FF6B00; color:#fff; transform:scale(1.1); }
.qty-control { display:flex; align-items:center; gap:6px; }
.qty-btn { background:transparent; border:2px solid #FFE600; border-radius:6px; width:28px; height:28px; cursor:pointer; color:#FFE600; font-size:1rem; font-weight:900; display:flex; align-items:center; justify-content:center; transition:all .15s; line-height:1; }
.qty-btn:hover { background:#FFE600; color:#000; }
.qty-num { font-family:'Anton',cursive; font-size:1rem; letter-spacing:1px; color:#FFE600; min-width:18px; text-align:center; }

/* ── MODAL DETALLE ── */
.modal-overlay { position:fixed; inset:0; background:rgba(0,0,0,.92); z-index:300; display:flex; align-items:center; justify-content:center; padding:20px; animation:fadeIn .2s ease; }
@keyframes fadeIn { from{opacity:0} to{opacity:1} }
.detail-modal { background:#111; border:2px solid #FFE600; border-radius:16px; width:100%; max-width:440px; overflow:hidden; animation:popIn .3s cubic-bezier(.34,1.56,.64,1); position:relative; }
@keyframes popIn { from{opacity:0;transform:scale(.85)} to{opacity:1;transform:scale(1)} }
.detail-close { position:absolute; top:12px; right:12px; background:rgba(0,0,0,.7); border:2px solid #FFE600; border-radius:6px; width:34px; height:34px; color:#FFE600; font-size:1rem; font-weight:900; cursor:pointer; display:flex; align-items:center; justify-content:center; z-index:2; transition:all .2s; }
.detail-close:hover { background:#E63946; border-color:#E63946; color:#fff; }
.detail-img-wrap { width:100%; aspect-ratio:4/3; background:#1a1a1a; overflow:hidden; display:flex; align-items:center; justify-content:center; }
.detail-img { width:100%; height:100%; object-fit:cover; display:block; }
.detail-body { padding:20px 22px 24px; }
.detail-name { font-family:'Anton',cursive; font-size:1.8rem; letter-spacing:2px; color:#FFE600; text-transform:uppercase; margin-bottom:10px; }
.detail-desc { color:#aaa; font-size:.95rem; line-height:1.6; margin-bottom:20px; }
.detail-footer { display:flex; align-items:center; justify-content:space-between; }
.detail-price { font-family:'Anton',cursive; font-size:2.2rem; letter-spacing:2px; color:#FFE600; }
.detail-add { background:#FF6B00; border:none; border-radius:10px; padding:12px 22px; color:#fff; font-family:'Anton',cursive; font-size:.95rem; letter-spacing:1px; cursor:pointer; transition:all .2s; }
.detail-add:hover { background:#FFE600; color:#000; transform:scale(1.05); }

/* ── CARRITO SIDEBAR ── */
.cart-overlay-bg { position:fixed; inset:0; background:rgba(0,0,0,.80); z-index:200; animation:fadeIn .2s ease; }
.cart-sidebar { position:fixed; top:0; right:0; width:400px; max-width:100vw; height:100vh; background:#050505; border-left:2px solid #FFE600; z-index:201; display:flex; flex-direction:column; animation:slideIn .3s ease; }
@keyframes slideIn { from{transform:translateX(100%)} to{transform:translateX(0)} }
.cart-head { padding:18px 20px; border-bottom:2px solid #FFE600; display:flex; align-items:center; justify-content:space-between; background:#000; }
.cart-head h2 { font-family:'Anton',cursive; font-size:1.6rem; letter-spacing:3px; color:#FFE600; text-transform:uppercase; }
.close-x { background:transparent; border:2px solid #333; border-radius:6px; width:34px; height:34px; color:#aaa; font-size:1rem; cursor:pointer; display:flex; align-items:center; justify-content:center; transition:all .2s; font-weight:900; }
.close-x:hover { border-color:#E63946; color:#E63946; }
.cart-items { flex:1; overflow-y:auto; padding:16px; scrollbar-width:thin; scrollbar-color:#FFE600 transparent; background:#050505; }
.cart-empty { text-align:center; padding:60px 20px; }
.cart-empty .ee { font-size:4rem; margin-bottom:14px; }
.cart-empty p { font-family:'Anton',cursive; font-size:1.2rem; letter-spacing:2px; color:#444; }
.cart-item { background:#111; border:1px solid #1e1e1e; border-radius:10px; padding:12px; display:flex; align-items:center; gap:10px; margin-bottom:10px; transition:border-color .2s; }
.cart-item:hover { border-color:#333; }
.ci-emoji { font-size:2rem; flex-shrink:0; }
.ci-info { flex:1; min-width:0; }
.ci-name { font-family:'Anton',cursive; font-size:.85rem; letter-spacing:1px; color:#FF6B00; text-transform:uppercase; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; margin-bottom:6px; }
.ci-bottom { display:flex; align-items:center; justify-content:space-between; }
.ci-total { font-family:'Anton',cursive; font-size:1rem; letter-spacing:1px; color:#FFE600; }
.ci-del { background:none; border:none; color:#333; cursor:pointer; font-size:1.1rem; padding:4px; border-radius:4px; transition:color .2s; flex-shrink:0; }
.ci-del:hover { color:#E63946; }
.cart-summary { padding:18px 20px; border-top:2px solid #FFE600; background:#000; }
.sum-row { display:flex; justify-content:space-between; font-size:.9rem; color:#555; margin-bottom:8px; font-weight:700; }
.sum-row.total { font-family:'Anton',cursive; font-size:1.4rem; letter-spacing:2px; color:#fff; border-top:1px solid #1a1a1a; padding-top:10px; margin-top:4px; }
.sum-row.total span:last-child { color:#FFE600; }
.checkout-btn { width:100%; background:#FF6B00; border:none; border-radius:10px; padding:15px; color:#fff; font-family:'Anton',cursive; font-size:1.1rem; letter-spacing:2px; cursor:pointer; margin-top:14px; transition:all .2s; text-transform:uppercase; }
.checkout-btn:hover:not(:disabled) { background:#FFE600; color:#000; transform:translateY(-1px); }
.checkout-btn:disabled { opacity:.35; cursor:not-allowed; }

/* ── CHECKOUT MODAL ── */
.checkout-modal { background:#080808; border:2px solid #FFE600; border-radius:16px; width:100%; max-width:500px; max-height:92vh; overflow-y:auto; animation:popIn .3s cubic-bezier(.34,1.56,.64,1); scrollbar-width:thin; scrollbar-color:#FFE600 transparent; }
.form-head { background:#000; border-bottom:2px solid #FFE600; padding:20px 24px; display:flex; align-items:center; justify-content:space-between; }
.form-title { font-family:'Anton',cursive; font-size:1.6rem; letter-spacing:3px; color:#FFE600; text-transform:uppercase; }
.form-body { padding:22px 24px; }
.form-group { margin-bottom:16px; }
.form-group label { display:block; font-family:'Anton',cursive; font-size:.8rem; letter-spacing:1.5px; color:#555; text-transform:uppercase; margin-bottom:7px; }
.form-group input,
.form-group textarea,
.form-group select { width:100%; background:#111; border:2px solid #222; border-radius:8px; padding:11px 14px; color:#fff; font-family:'Nunito',sans-serif; font-size:.95rem; outline:none; transition:all .25s; resize:none; }
.form-group input:focus,
.form-group textarea:focus,
.form-group select:focus { border-color:#FFE600; background:#0d0d0d; }
.form-group input.error,
.form-group textarea.error,
.form-group select.error { border-color:#E63946; }
.form-group select { appearance:none; cursor:pointer; background-color:#111; }
.form-group select option { background:#111; }
.form-group input::placeholder,
.form-group textarea::placeholder { color:#333; }
.error-msg { font-size:.76rem; color:#E63946; font-weight:800; margin-top:5px; }
.form-row { display:grid; grid-template-columns:1fr 1fr; gap:12px; }
.form-actions { padding:0 24px 24px; display:flex; gap:10px; flex-wrap:wrap; }
.btn-back { flex:1; background:transparent; border:2px solid #222; border-radius:10px; padding:13px; color:#666; font-family:'Anton',cursive; font-size:.95rem; letter-spacing:1px; cursor:pointer; transition:all .2s; min-width:80px; }
.btn-back:hover { border-color:#555; color:#fff; }
.btn-confirm { flex:2; background:#FF6B00; border:none; border-radius:10px; padding:13px; color:#fff; font-family:'Anton',cursive; font-size:.95rem; letter-spacing:2px; cursor:pointer; transition:all .2s; text-transform:uppercase; min-width:120px; }
.btn-confirm:hover:not(:disabled) { background:#FFE600; color:#000; }
.btn-confirm:disabled { opacity:.4; cursor:not-allowed; }
.btn-download { flex:1.5; background:transparent; border:2px solid #FFE600; border-radius:10px; padding:13px; color:#FFE600; font-family:'Anton',cursive; font-size:.9rem; letter-spacing:1px; cursor:pointer; transition:all .2s; min-width:100px; text-transform:uppercase; }
.btn-download:hover { background:#FFE600; color:#000; }

/* ── FACTURA ── */
.invoice-wrap { padding:0 24px 24px; }
.inv-head-bar { padding:20px 24px; display:flex; align-items:center; justify-content:space-between; border-bottom:2px solid #FFE600; background:#000; }
.invoice { background:#fff; color:#111; border-radius:10px; overflow:hidden; margin-top:22px; }
.inv-top { background:#000; padding:18px 20px; text-align:center; border-bottom:3px solid #FFE600; }
.inv-logo-text { font-family:'Permanent Marker',cursive; font-size:1.8rem; background:linear-gradient(180deg,#FFD700 0%,#FF6B00 60%,#FF3300 100%); -webkit-background-clip:text; -webkit-text-fill-color:transparent; background-clip:text; letter-spacing:2px; }
.inv-sub { font-size:.75rem; color:#666; margin-top:4px; }
.inv-meta { display:grid; grid-template-columns:1fr 1fr; gap:1px; background:#eee; border-bottom:1px solid #ddd; }
.inv-cell { background:#fafafa; padding:10px 14px; }
.inv-cell label { display:block; font-size:.68rem; text-transform:uppercase; letter-spacing:.5px; color:#999; font-weight:700; margin-bottom:2px; }
.inv-cell span { font-size:.85rem; font-weight:800; color:#111; }
.inv-cell.wide { grid-column:span 2; }
.inv-table-wrap { padding:14px 16px; }
.inv-table-wrap table { width:100%; border-collapse:collapse; font-size:.82rem; }
.inv-table-wrap th { text-align:left; padding:7px 0; border-bottom:2px solid #111; font-size:.72rem; text-transform:uppercase; letter-spacing:.5px; color:#333; }
.inv-table-wrap th:not(:first-child) { text-align:center; }
.inv-table-wrap th:last-child { text-align:right; }
.inv-table-wrap td { padding:7px 0; border-bottom:1px solid #eee; color:#111; }
.inv-table-wrap td:not(:first-child) { text-align:center; }
.inv-table-wrap td:last-child { text-align:right; font-weight:800; }
.inv-totals { background:#f5f5f5; padding:12px 16px; }
.inv-tot { display:flex; justify-content:space-between; font-size:.85rem; color:#555; padding:3px 0; font-weight:600; }
.inv-tot.grand { border-top:2px solid #111; margin-top:6px; padding-top:8px; font-size:1.1rem; font-weight:900; color:#111; }
.inv-foot-note { background:#000; color:#555; text-align:center; padding:12px; font-size:.75rem; font-family:'Anton',cursive; letter-spacing:2px; }

/* ── HISTORIAL ── */
.history-modal { background:#080808; border:2px solid #FFE600; border-radius:16px; width:100%; max-width:560px; max-height:88vh; overflow:hidden; display:flex; flex-direction:column; animation:popIn .3s cubic-bezier(.34,1.56,.64,1); }
.history-body { flex:1; overflow-y:auto; padding:16px 20px; scrollbar-width:thin; scrollbar-color:#FFE600 transparent; }
.history-item { background:#111; border:1px solid #1e1e1e; border-radius:10px; padding:14px 16px; margin-bottom:12px; transition:border-color .2s; }
.history-item:hover { border-color:#333; }
.history-item-head { display:flex; align-items:center; justify-content:space-between; margin-bottom:6px; }
.history-num { font-family:'Anton',cursive; font-size:.95rem; letter-spacing:2px; color:#FFE600; margin-right:10px; }
.history-date { font-size:.78rem; color:#555; font-weight:700; }
.history-total { font-family:'Anton',cursive; font-size:1.1rem; color:#FF6B00; }
.history-client { font-size:.82rem; color:#777; margin-bottom:8px; font-weight:700; }
.history-products { display:flex; flex-wrap:wrap; gap:6px; margin-bottom:8px; }
.h-prod-chip { background:#1a1a1a; border:1px solid #2a2a2a; border-radius:20px; padding:3px 10px; font-size:.74rem; color:#aaa; font-weight:700; }
.h-tag { background:#1a1a1a; border:1px solid #333; border-radius:4px; padding:2px 8px; font-size:.72rem; color:#666; font-weight:800; font-family:'Anton',cursive; letter-spacing:1px; }
.btn-history-pdf { background:transparent; border:1.5px solid #FFE600; border-radius:6px; padding:3px 10px; color:#FFE600; font-family:'Anton',cursive; font-size:.72rem; letter-spacing:1px; cursor:pointer; transition:all .2s; }
.btn-history-pdf:hover { background:#FFE600; color:#000; }

/* ── PANEL ADMIN ── */
.product-preview { background:#111; border:1px dashed #333; border-radius:8px; padding:14px; margin-top:4px; }

/* ── SPINNER ── */
.spinner-overlay { position:fixed; inset:0; background:rgba(0,0,0,.96); z-index:500; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:22px; }
.spinner { width:64px; height:64px; border:5px solid #1a1a1a; border-top-color:#FFE600; border-right-color:#FF6B00; border-radius:50%; animation:spin .7s linear infinite; }
@keyframes spin { to{transform:rotate(360deg)} }
.spinner-label { font-family:'Anton',cursive; font-size:1.4rem; letter-spacing:4px; color:#FFE600; animation:blink 1s ease-in-out infinite; }
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:.3} }

/* ── NOTIFICACIONES ── */
.notifs { position:fixed; top:86px; right:20px; z-index:999; display:flex; flex-direction:column; gap:8px; pointer-events:none; }
.notif { background:#0d0d0d; border-left:4px solid #FFE600; border-radius:8px; padding:11px 16px; font-family:'Nunito',sans-serif; font-weight:800; font-size:.88rem; display:flex; align-items:center; gap:8px; min-width:240px; color:#fff; box-shadow:0 4px 20px rgba(0,0,0,.8); pointer-events:all; animation:nIn .35s cubic-bezier(.34,1.56,.64,1); }
.notif.success { border-left-color:#00e676; }
.notif.error   { border-left-color:#E63946; }
.notif.info    { border-left-color:#FF6B00; }
@keyframes nIn { from{opacity:0;transform:translateX(80px)} to{opacity:1;transform:translateX(0)} }

/* ── ÉXITO ── */
.success-box { padding:40px 24px; text-align:center; }
.success-big { font-size:5rem; margin-bottom:16px; animation:popIn .5s ease; }
.success-box h2 { font-family:'Anton',cursive; font-size:2rem; letter-spacing:4px; color:#00e676; margin-bottom:8px; text-transform:uppercase; }
.success-box p { color:#666; font-size:.95rem; margin-bottom:10px; }
.order-num { border:2px solid #FFE600; border-radius:8px; padding:12px 20px; font-family:'Anton',cursive; font-size:1.6rem; letter-spacing:4px; color:#FFE600; margin:18px 0; display:inline-block; }

/* ── SIN RESULTADOS ── */
.no-results { text-align:center; padding:60px 20px; }
.no-results p { font-family:'Anton',cursive; font-size:1.4rem; letter-spacing:2px; color:#222; text-transform:uppercase; margin-top:12px; }

/* ── FOOTER ── */
footer { background:#000; border-top:3px solid #FFE600; padding:28px 20px; text-align:center; }
.footer-logo { font-family:'Permanent Marker',cursive; font-size:1.6rem; background:linear-gradient(180deg,#FFD700 0%,#FF6B00 100%); -webkit-background-clip:text; -webkit-text-fill-color:transparent; background-clip:text; margin-bottom:8px; }
.footer-info { color:#444; font-size:.82rem; font-weight:700; }

/* ── TRANSICIONES ── */
.fade-enter-active,.fade-leave-active { transition:opacity .2s; }
.fade-enter-from,.fade-leave-to       { opacity:0; }
.slide-enter-active,.slide-leave-active { transition:transform .3s ease; }
.slide-enter-from,.slide-leave-to      { transform:translateX(100%); }
.notif-enter-active,.notif-leave-active { transition:all .3s ease; }
.notif-enter-from { opacity:0; transform:translateX(80px); }
.notif-leave-to   { opacity:0; transform:translateX(80px); }

/* ── RESPONSIVE ── */
@media(max-width:600px){
  .logo-text { font-size:1.5rem; }
  .cart-btn span:not(.cart-badge) { display:none; }
  .cart-sidebar { width:100vw; }
  .product-grid { grid-template-columns:repeat(2,1fr); gap:12px; }
  .card-name { font-size:.88rem; }
  .price { font-size:1.1rem; }
  .form-row { grid-template-columns:1fr; }
  .form-actions { flex-wrap:wrap; }
  .btn-download { flex:100%; }
}
</style>
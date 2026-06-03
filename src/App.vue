
<template>
<div class="container">

  <header class="header">

    <h1>🍔 Restaurante Vue Food</h1>

    <div style="display: flex; align-items: center; gap: 15px;">
      <button 
        @click="mostrarFormulario = !mostrarFormulario" 
        style="background: white; color: #ff5f6d; border: none; padding: 10px 18px; border-radius: 50px; font-weight: 700; cursor: pointer;"
      >
        {{ mostrarFormulario ? '✕ Cerrar Formulario' : '➕ Nuevo Producto' }}
      </button>

      <div class="carrito" @click="mostrarCarrito = true">
        🛒 {{ carrito.length }}
      </div>
    </div>

  </header>

  <div class="filtros">

    <input
      v-model="busqueda"
      placeholder="Buscar producto..."
      class="input-busqueda"
    />

    <select v-model="categoriaSeleccionada">

      <option value="">
        Todas las categorías
      </option>

      <option
        v-for="cat in categorias"
        :key="cat"
        :value="cat"
      >
        {{ cat }}
      </option>

    </select>

  </div>

  <div v-if="mostrarFormulario" class="formulario" style="background: white; padding: 20px; border-radius: 15px; box-shadow: 0 4px 15px rgba(0,0,0,0.05);">

    <input
      v-model="nuevo.nombre"
      placeholder="Nombre"
    />

    <input
      v-model="nuevo.descripcion"
      placeholder="Descripción"
    />

    <input
      v-model.number="nuevo.precio"
      type="number"
      placeholder="Precio"
    />

    <input
      v-model.number="nuevo.stock"
      type="number"
      placeholder="Stock"
    />

    <input
      v-model="nuevo.categoria"
      placeholder="Categoría"
    />

    <input
      v-model="nuevo.imagen"
      placeholder="URL Imagen"
    />

    <button @click="agregarProducto">
      Agregar Producto
    </button>

  </div>

  <div class="productos">

    <div
      class="card"
      v-for="producto in productosFiltrados"
      :key="producto.id"
    >

      <img :src="producto.imagen" />

      <h3>{{ producto.nombre }}</h3>

      <p>{{ producto.descripcion }}</p>

      <p class="categoria">
        {{ producto.categoria }}
      </p>

      <h4>
        {{ formatearMoneda(producto.precio) }}
      </h4>

      <p class="stock">
        Stock: {{ producto.stock }}
      </p>

      <div class="botones-card">

        <button
          @click="agregarCarrito(producto)"
          :disabled="producto.stock <= 0"
        >
          {{ producto.stock <= 0 ? 'Sin Stock' : 'Agregar' }}
        </button>

        <button
          class="btn-stock"
          @click="sumarStock(producto)"
        >
          + Stock
        </button>

      </div>

    </div>

  </div>

  <div
    v-if="mostrarCarrito"
    class="panel-carrito"
  >

    <div class="carrito-header">

      <h2>Carrito</h2>

      <button
        class="cerrar-carrito"
        @click="mostrarCarrito = false"
      >
        ✕
      </button>

    </div>

    <div v-if="carrito.length === 0">

      <p>
        Tu carrito está vacío
      </p>

    </div>

    <div
      v-for="item in carrito"
      :key="item.id"
      class="item"
    >

      <span>{{ item.nombre }}</span>

      <span>x {{ item.cantidad }}</span>

      <span>
        {{ formatearMoneda(item.precio * item.cantidad) }}
      </span>

      <button @click="eliminar(item)">
        ❌
      </button>

    </div>

    <h3 class="total">
      Total: {{ formatearMoneda(total) }}
    </h3>

    <button
      class="finalizar"
      @click="finalizarCompra"
    >
      Finalizar Compra
    </button>

  </div>

  <div
    v-if="loading"
    class="spinner"
  >
    Procesando compra...
  </div>

  <div
    v-if="notificacion"
    class="notificacion"
  >
    {{ notificacion }}
  </div>

</div>
</template>

<script setup>
import { ref, computed } from 'vue'
import jsPDF from "jspdf"

const mostrarCarrito = ref(false)
const mostrarFormulario = ref(false) // <- NUEVO ESTADO AGREGADO
const loading = ref(false)
const notificacion = ref('')

const busqueda = ref('')
const categoriaSeleccionada = ref('')

const productos = ref([

{
id:1,
nombre:'Hamburguesa Clásica',
categoria:'Hamburguesas',
descripcion:'Carne y queso',
precio:18000,
stock:10,
imagen:'https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg'
},

{
id:2,
nombre:'Hamburguesa Doble',
categoria:'Hamburguesas',
descripcion:'Doble carne',
precio:22000,
stock:10,
imagen:'https://progcarne.com/storage/app/uploads/public/608/6d1/8b0/6086d18b065a7811052900.jpg'
},

{
id:3,
nombre:'Pizza Pepperoni',
categoria:'Pizzas',
descripcion:'Pizza pepperoni',
precio:23000,
stock:10,
imagen:'https://ik.imagekit.io/smithfield/armour/4353bced-f940-00d0-8c6e-13a0a4a7f5c2/2ac60829-5178-4a6e-80cf-6ca43d862cee/Quick-and-Easy-Pepperoni-Pizza-700x700.jpeg'
},

{
id:4,
nombre:'Gaseosa',
categoria:'Bebidas',
descripcion:'Bebida fría',
precio:5000,
stock:10,
imagen:'https://www.aguaparatuvida.com.ar/img/novedades/106.jpg'
},

{
id:5,
nombre:'Brownie',
categoria:'Postres',
descripcion:'Brownie de chocolate',
precio:7000,
stock:10,
imagen:'https://www.cocinista.es/download/bancorecursos/recetas/receta-brownies.jpg'
}

])

const carrito = ref([])

const nuevo = ref({
nombre:'',
descripcion:'',
precio:null,
stock:null,
categoria:'',
imagen:''
})

/* =========================
   CATEGORÍAS
========================= */

const categorias = computed(()=>{

  return [
    ...new Set(
      productos.value.map(
        p => p.categoria
      )
    )
  ]

})

/* =========================
   FILTRO PRODUCTOS
========================= */

const productosFiltrados = computed(()=>{

  return productos.value.filter(producto=>{

    const coincideBusqueda =
      producto.nombre
      .toLowerCase()
      .includes(
        busqueda.value.toLowerCase()
      )

    const coincideCategoria =
      categoriaSeleccionada.value === '' ||
      producto.categoria === categoriaSeleccionada.value

    return coincideBusqueda &&
           coincideCategoria

  })

})

/* =========================
   FORMATEAR MONEDA
========================= */

const formatearMoneda = (valor)=>{

  return new Intl.NumberFormat(
    'es-CO',
    {
      style:'currency',
      currency:'COP'
    }
  ).format(valor)

}

/* =========================
   AGREGAR CARRITO
========================= */

const agregarCarrito = (producto)=>{

  if(producto.stock <= 0){

    notificacion.value = 'Sin stock'

    return
  }

  const existe = carrito.value.find(
    p => p.id === producto.id
  )

  if(existe){

    existe.cantidad++

  }else{

    carrito.value.push({
      ...producto,
      cantidad:1
    })

  }

  producto.stock--

  notificacion.value = 'Producto agregado'

  setTimeout(()=>{

    notificacion.value = ''

  },2000)

}


const sumarStock = (producto)=>{

  producto.stock++

  notificacion.value = 'Stock agregado'

  setTimeout(()=>{ 

    notificacion.value = ''

  },1500)

}


const eliminar = (item)=>{

  const prod = productos.value.find(
    p => p.id === item.id
  )

  if(prod){

    prod.stock += item.cantidad

  }

  carrito.value = carrito.value.filter(
    p => p.id !== item.id
  )

  notificacion.value = 'Producto eliminado'

  setTimeout(()=>{

    notificacion.value = ''

  },2000)

}

const total = computed(()=>{

  return carrito.value.reduce(
    (acc,item)=>{

      return acc +
      item.precio * item.cantidad

    },0
  )

})



const agregarProducto = ()=>{

  if(
    !nuevo.value.nombre ||
    !nuevo.value.precio
  ){

    notificacion.value =
    'Complete los campos'

    return
  }

  productos.value.push({

    id: productos.value.length + 1,

    nombre:nuevo.value.nombre,
    descripcion:nuevo.value.descripcion,
    precio:nuevo.value.precio,
    stock:nuevo.value.stock,
    categoria:nuevo.value.categoria,
    imagen:nuevo.value.imagen

  })

  nuevo.value = {

    nombre:'',
    descripcion:'',
    precio:null,
    stock:null,
    categoria:'',
    imagen:''

  }

  notificacion.value =
  'Producto agregado'
  
  // Cerramos el formulario automáticamente después de agregar con éxito
  mostrarFormulario.value = false 

  setTimeout(()=>{

    notificacion.value = ''

  },2000)

}


const generarPDF = ()=>{

  const doc = new jsPDF()

  doc.setFontSize(20)

  doc.text('FACTURA',85,20)

  doc.setFontSize(12)

  doc.text(
    'Restaurante Vue Food',
    20,
    35
  )

  doc.text(
    'NIT: 900123456',
    20,
    42
  )

  doc.text(
    'San Gil - Santander',
    20,
    49
  )

  doc.line(20,55,190,55)

  let y = 68

  carrito.value.forEach(item=>{

    doc.text(

      `${item.nombre} x${item.cantidad} ${formatearMoneda(item.precio * item.cantidad)}`,

      20,

      y

    )

    y += 10

  })

  doc.line(20,y,190,y)

  y += 12

  doc.text(

    `TOTAL: ${formatearMoneda(total.value)}`,

    20,

    y

  )

  y += 15

  doc.text(
    'Gracias por su compra',
    20,
    y
  )

  doc.output('dataurlnewwindow')

}

/* =========================
   FINALIZAR COMPRA
========================= */

const finalizarCompra = ()=>{

  if(carrito.value.length === 0){

    notificacion.value =
    'Carrito vacío'

    return
  }

  loading.value = true

  setTimeout(()=>{

    loading.value = false

    mostrarCarrito.value = false

    generarPDF()

    carrito.value = []

    notificacion.value =
    'Compra exitosa'

    setTimeout(()=>{

      notificacion.value = ''

    },2000)

  },1500)

}
</script>

<style>

body{
margin:0;
padding:0;
background:linear-gradient(
135deg,
#fff8f0,
#ffe8e8,
#eef7ff
);
font-family:'Poppins','Segoe UI',sans-serif;
}

.container{
padding:25px;
max-width:1400px;
margin:auto;
}


.header{
display:flex;
justify-content:space-between;
align-items:center;
padding:18px 25px;
border-radius:20px;
background:linear-gradient(
135deg,
#ff5f6d,
#ffc371
);
color:white;
}

.carrito{
background:white;
color:#ff5f6d;
padding:10px 18px;
border-radius:50px;
font-weight:700;
cursor:pointer;
}



.filtros,
.formulario{
display:flex;
gap:10px;
margin-top:20px;
flex-wrap:wrap;
}

.input-busqueda,
.formulario input,
select{
padding:12px;
border-radius:10px;
border:1px solid #ddd;
flex:1;
}

.formulario button{
border:none;
padding:12px 20px;
border-radius:10px;
background:#ff5f6d;
color:white;
cursor:pointer;
}



.productos{
display:grid;
grid-template-columns:
repeat(3,1fr);
gap:25px;
margin-top:30px;
}

.card{
background:white;
border-radius:22px;
overflow:hidden;
box-shadow:
0 12px 30px rgba(0,0,0,.08);
transition:.3s;
}

.card:hover{
transform:translateY(-5px);
}

.card img{
width:100%;
height:220px;
object-fit:cover;
}

.card h3,
.card p,
.card h4,
.stock,
.categoria{
margin-left:15px;
margin-right:15px;
}

.categoria{
font-size:14px;
color:#2196f3;
font-weight:700;
}

.stock{
font-weight:700;
color:#ff5f6d;
}

.botones-card{
padding:0 15px 18px;
display:flex;
gap:10px;
}

.card button{
flex:1;
border:none;
padding:12px;
border-radius:12px;
background:linear-gradient(
135deg,
#00c853,
#64dd17
);
color:white;
cursor:pointer;
font-weight:700;
}

.card button:disabled{
background:#999;
cursor:not-allowed;
opacity:.6;
}

.btn-stock{
background:linear-gradient(
135deg,
#2196f3,
#00b0ff
)!important;
}



.panel-carrito{
position:fixed;
right:0;
top:0;
width:360px;
height:100%;
background:white;
padding:25px;
overflow:auto;
z-index:999;
box-shadow:
-5px 0 20px rgba(0,0,0,.15);
}

.carrito-header{
display:flex;
justify-content:space-between;
align-items:center;
}

.cerrar-carrito{
width:42px;
height:42px;
border:none;
border-radius:50%;
background:#ff5f6d;
color:white;
cursor:pointer;
font-size:18px;
}

.item{
display:grid;
grid-template-columns:
1fr auto auto auto;
gap:8px;
padding:10px;
background:#f8f8f8;
margin-bottom:10px;
border-radius:10px;
align-items:center;
}

.total{
margin-top:20px;
}

.finalizar{
width:100%;
padding:14px;
border:none;
border-radius:14px;
background:linear-gradient(
135deg,
#ff5f6d,
#ff9966
);
color:white;
cursor:pointer;
font-weight:700;
}



.spinner{
position:fixed;
top:50%;
left:50%;
transform:translate(-50%,-50%);
background:#222;
color:white;
padding:18px 28px;
border-radius:14px;
z-index:2000;
}


.notificacion{
position:fixed;
bottom:20px;
right:20px;
background:#00c853;
color:white;
padding:14px 22px;
border-radius:50px;
z-index:3000;
}



@media(max-width:900px){

.productos{
grid-template-columns:
repeat(2,1fr);
}

}

@media(max-width:768px){

.header{
flex-direction:column;
gap:15px;
text-align:center;
}

.productos{
grid-template-columns:1fr;
}

.panel-carrito{
width:100%;
padding:15px;
}

.item{
grid-template-columns:1fr;
text-align:center;
}

.botones-card{
flex-direction:column;
}

}

</style>


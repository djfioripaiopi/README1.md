# README1.md

# Mini Sales Automation Dashboard 

Core Architecture

Modular feature-based structure (auth, inventory, sales)

Vue 3 Composition API + TypeScript

Pinia for centralized state management

Component Design

Container/presentational component separation

Smart reusable components with strict props

Layout components for consistent UI structure

Data Flow

Unidirectional: Mock API → Stores → Components

Stores handle all business logic

Simulated loading states/errors

Performance

Lazy-loaded routes

Virtual scrolling for large lists

Memoized computations

Responsive CSS with mobile-first approach

Key Libraries

PrimeVue for UI components

Chart.js for data visualization

Leaflet for interactive maps

Vuelidate for form validation

Testing

Vitest for unit tests

Testing Library for components

Focus on stores and complex logic

Trade-offs

PrimeVue over custom components for speed

Client-side filtering instead of server-side

Basic error handling to meet deadlines

This architecture prioritizes:

Maintainability through clear separation of concerns

Scalability via modular design

Type safety with TypeScript

Performance with smart loading strategies

Testability through isolated units

1. Authentication Module
Login Screen Components:
<template>
  <Card class="login-card">
    <form @submit.prevent="handleLogin">
      <InputText v-model="email" placeholder="Email" />
      <Password v-model="password" toggleMask :feedback="false" />
      <Checkbox v-model="rememberMe" binary label="Remember me" />
      <Button type="submit" :disabled="!isFormValid" :loading="loading">
        Login
      </Button>
      <a @click="showResetForm">Forgot password?</a>
    </form>
  </Card>
</template>

<script setup>

const passwordRules = [
  (v) => v.length >= 8 || 'Min 8 characters',
  (v) => /[A-Z]/.test(v) || '1 uppercase letter',
  (v) => /\d/.test(v) || '1 number',
  (v) => /[!@#$%^&*]/.test(v) || '1 special character'
];
</script>
Auth Store Highlights:

const login = async (credentials) => {
  try {
    loading.value = true;
    const user = mockUsers.find(u => u.email === credentials.email);
    
    if (!user || !verifyPassword(credentials.password, user.password)) {
      throw new Error('Invalid credentials');
    }
    
    user.value = user;
    if (credentials.rememberMe) {
      localStorage.setItem('authToken', generateMockToken(user));
    } else {
      sessionStorage.setItem('authToken', generateMockToken(user));
    }
  } finally {
    loading.value = false;
  }
};

2. Dashboard Overview
Responsive Layout Structure:
<template>
  <div class="dashboard-layout">
    <AppHeader />
    <Sidebar :collapsed="isCollapsed" />
    
    <main class="content-area">
      <Breadcrumbs />
      
      <div class="grid">
        <SalesChart class="col-12 md:col-8" />
        <InventoryStatus class="col-12 md:col-4" />
        <TopProducts class="col-12" />
      </div>
    </main>
  </div>
</template>

<style>
.dashboard-layout {
  display: grid;
  grid-template-columns: auto 1fr;
  grid-template-rows: auto 1fr;
}

@media (max-width: 768px) {
  .dashboard-layout {
    grid-template-columns: 1fr;
  }
}
</style>
Chart Implementation:
// components/SalesChart.vue
const chartOptions = ref({
  responsive: true,
  plugins: {
    tooltip: {
      callbacks: {
        label: (ctx) => `$${ctx.raw.toLocaleString()}`
      }
    }
  }
});

const fetchData = async (range) => {
  const res = await import(`@/data/sales-${range}.json`);
  chartData.value = transformData(res.default);
};

3. Inventory Management
Data Table with Virtual Scroll:
<DataTable
  :value="filteredProducts"
  :rows="20"
  scrollable
  scrollHeight="flex"
  virtualScroller
>
  <Column field="sku" header="SKU" sortable />
  <Column field="name" header="Name" sortable />
  <Column field="stock" header="Stock" sortable>
    <template #body="{ data }">
      <InventoryBadge :value="data.stock" />
    </template>
  </Column>
</DataTable>

Product Detail Tabs:
<TabView>
  <TabPanel header="Overview">
    <ProductImages :images="product.images" />
    <ProductSpecs :specs="product.specs" />
  </TabPanel>
  <TabPanel header="Inventory">
    <WarehouseTable :stock="product.warehouses" />
  </TabPanel>
</TabView>

4. Sales Transaction Module
Order Creation Flow:
<Stepper :model="steps" :activeIndex="currentStep">
  <template #item="{ item }">
    <span :class="{ 'text-primary': currentStep >= item.index }">
      {{ item.label }}
    </span>
  </template>
</Stepper>

<CustomerSearch v-if="currentStep === 0" @select="setCustomer" />
<ProductSelection v-if="currentStep === 1" @update="updateCart" />
<OrderSummary v-if="currentStep === 2" :items="cart" />
Real-time Stock Check:
const checkStock = debounce(async (productId, quantity) => {
  const available = await inventoryStore.checkStock(productId);
  if (quantity > available) {
    showWarning(`Only ${available} units available`);
  }
}, 300);

5. Customer Map View
Leaflet Implementation:
// components/CustomerMap.vue
onMounted(() => {
  map.value = L.map('map').setView([51.505, -0.09], 3);
  
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map.value);

  customers.value.forEach(c => {
    L.marker([c.lat, c.lng])
     .bindPopup(popupContent(c))
     .addTo(map.value);
  });
});

6. Notifications System
Toast Service:
// composables/useToast.ts
export function useToast() {
  const toast = useToast();
  
  return {
    success: (msg) => toast.add({
      severity: 'success',
      summary: 'Success',
      detail: msg,
      life: 3000
    }),
    error: (msg) => toast.add({
      severity: 'error', 
      summary: 'Error',
      detail: msg,
      life: 5000
    })
  };
}
Notification Center:
<OverlayPanel ref="panel">
  <div class="notification-item" 
       v-for="note in unread" 
       :key="note.id"
       @click="markRead(note.id)">
    <i :class="`icon-${note.type}`" />
    <div class="content">
      <h5>{{ note.title }}</h5>
      <p>{{ truncate(note.message) }}</p>
    </div>
  </div>
</OverlayPanel>



MOCK DATA AND RESOURCES 
1. user_data.json
[
  {
    "user_id": 1,
    "name": "John Doe",
    "role": "sales_rep",
    "region": "Nairobi",
    "permissions": ["view_sales", "edit_profile"],
    "preferences": {
      "theme": "dark",
      "notifications": true,
      "dashboard_widgets": ["sales", "inventory"]
    }
  },
  {
    "user_id": 2,
    "name": "Jane Smith",
    "role": "manager",
    "region": "Mombasa",
    "permissions": ["view_sales", "approve_orders", "manage_team"],
    "preferences": {
      "theme": "light",
      "notifications": false,
      "dashboard_widgets": ["sales", "customers", "inventory"]
    }
  }
]

2. products.json
[
  {
    "product_id": "P001",
    "name": "Super Motor Oil 5W-30",
    "category": "Lubricants",
    "subcategory": "Engine Oil",
    "brand": "OilMax",
    "stock_quantity": 120,
    "price": 3500
  },
  {
    "product_id": "P002",
    "name": "Brake Fluid DOT 4",
    "category": "Fluids",
    "subcategory": "Brake Fluid",
    "brand": "BrakeSafe",
    "stock_quantity": 80,
    "price": 800
  }
]

3. customers.json
[
  {
    "customer_id": "C001",
    "name": "AutoZone Ltd.",
    "type": "retailer",
    "region": "Nairobi",
    "credit_limit": 100000,
    "balance": 25000
  },
  {
    "customer_id": "C002",
    "name": "Speedy Garage",
    "type": "distributor",
    "region": "Mombasa",
    "credit_limit": 150000,
    "balance": 100000
  }
]

4. sales_data.json
[
  {
    "sale_id": "S001",
    "customer_id": "C001",
    "total_amount": 3800,
    "status": "Delivered",
    "date": "2024-04-15"
  },
  {
    "sale_id": "S002",
    "customer_id": "C002",
    "total_amount": 16000,
    "status": "Processing",
    "date": "2024-04-16"
  }
]

## Acknowledgements

 - [Awesome Readme Templates](https://awesomeopensource.com/project/elangosundar/awesome-README-templates)
 - [Awesome README](https://github.com/matiassingers/awesome-readme)
 - [How to write a Good readme](https://bulldogjob.com/news/449-how-to-write-a-good-readme-for-your-github-project)


## API Reference

#### Get all items

```http
  GET /api/items
```

| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `api_key` | `string` | **Required**. Your API key |

#### Get item

```http
  GET /api/items/${id}
```

| Parameter | Type     | Description                       |
| :-------- | :------- | :-------------------------------- |
| `id`      | `string` | **Required**. Id of item to fetch |

#### add(num1, num2)

Takes two numbers and returns the sum.

"# Readme.md" 

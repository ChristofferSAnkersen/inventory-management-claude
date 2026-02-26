<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Create mode -->
            <template v-if="mode === 'create'">
              <div class="item-header">
                <div class="item-info">
                  <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                  <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
                </div>
              </div>

              <div class="summary-grid">
                <div class="summary-card">
                  <div class="summary-label">Quantity Needed</div>
                  <div class="summary-value">{{ backlogItem.quantity_needed }} units</div>
                </div>
                <div class="summary-card shortage">
                  <div class="summary-label">Shortage</div>
                  <div class="summary-value">{{ shortage }} units</div>
                </div>
              </div>

              <form class="po-form" @submit.prevent="submitForm">
                <div class="form-group">
                  <label class="form-label" for="supplier-name">Supplier Name</label>
                  <input
                    id="supplier-name"
                    v-model="form.supplierName"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity to Order</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="delivery-date">Expected Delivery Date</label>
                  <input
                    id="delivery-date"
                    v-model="form.deliveryDate"
                    type="date"
                    class="form-input"
                    required
                  />
                </div>

                <div v-if="formError" class="form-error">{{ formError }}</div>
              </form>
            </template>

            <!-- View mode -->
            <template v-else>
              <div class="po-id-row">
                <div class="info-label">Purchase Order ID</div>
                <div class="po-id">{{ poId }}</div>
              </div>

              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">Item Name</div>
                  <div class="info-value">{{ backlogItem.item_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Item SKU</div>
                  <div class="info-value sku">{{ backlogItem.item_sku }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Order ID</div>
                  <div class="info-value order-id">{{ backlogItem.order_id }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge placed">Order Placed</span>
                  </div>
                </div>
              </div>
            </template>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">
              {{ mode === 'create' ? 'Cancel' : 'Close' }}
            </button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              type="button"
              @click="submitForm"
            >
              Create Order
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create',
    validator: (val) => ['create', 'view'].includes(val)
  }
})

const emit = defineEmits(['close', 'po-created'])

const form = ref({
  supplierName: '',
  quantity: 0,
  deliveryDate: ''
})

const formError = ref(null)

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
})

const poId = computed(() => {
  if (!props.backlogItem) return ''
  return props.backlogItem.purchase_order_id || props.backlogItem.purchase_order?.id || 'N/A'
})

watch(() => [props.isOpen, props.backlogItem], ([open]) => {
  if (open && props.mode === 'create') {
    form.value = {
      supplierName: '',
      quantity: shortage.value > 0 ? shortage.value : 1,
      deliveryDate: ''
    }
    formError.value = null
  }
})

const close = () => {
  emit('close')
}

const submitForm = () => {
  formError.value = null

  if (!form.value.supplierName.trim()) {
    formError.value = 'Supplier name is required.'
    return
  }
  if (!form.value.quantity || form.value.quantity < 1) {
    formError.value = 'Quantity must be at least 1.'
    return
  }
  if (!form.value.deliveryDate) {
    formError.value = 'Expected delivery date is required.'
    return
  }

  emit('po-created', {
    id: 'PO-' + Date.now(),
    backlog_item_id: props.backlogItem?.id
  })
  close()
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 560px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

/* Create mode — item header */
.item-header {
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-name {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.375rem 0;
}

.item-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Summary cards */
.summary-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  margin-bottom: 2rem;
}

.summary-card {
  padding: 1.25rem;
  border-radius: 10px;
  border: 2px solid #e2e8f0;
  background: #f8fafc;
}

.summary-card.shortage {
  border-color: #fecaca;
  background: #fef2f2;
}

.summary-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.summary-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-card.shortage .summary-value {
  color: #dc2626;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  font-family: inherit;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.form-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode */
.po-id-row {
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.po-id {
  font-size: 1.25rem;
  font-weight: 700;
  color: #2563eb;
  font-family: 'Monaco', 'Courier New', monospace;
  margin-top: 0.375rem;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.order-id,
.info-value.sku {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.badge {
  display: inline-block;
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
}

.badge.placed {
  background: #dbeafe;
  color: #1e40af;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover {
  background: #1d4ed8;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>

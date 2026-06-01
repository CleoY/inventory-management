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
            <!-- Backlog item context -->
            <div class="context-banner">
              <div class="context-details">
                <span class="context-item-name">{{ backlogItem.item_name }}</span>
                <span class="context-sku">SKU: {{ backlogItem.item_sku }}</span>
              </div>
              <span class="shortage-pill">
                {{ Math.abs(backlogItem.quantity_needed - backlogItem.quantity_available) }} units short
              </span>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" @submit.prevent="submitForm" class="po-form">
              <div class="form-group">
                <label class="form-label" for="supplier_name">Supplier Name</label>
                <input
                  id="supplier_name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  required
                  placeholder="Enter supplier name"
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    required
                    min="1"
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit_cost">Unit Cost ($)</label>
                  <input
                    id="unit_cost"
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    required
                    min="0"
                    step="0.01"
                    placeholder="0.00"
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label" for="expected_delivery_date">Expected Delivery Date</label>
                <input
                  id="expected_delivery_date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="notes">Notes <span class="optional-label">(optional)</span></label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-textarea"
                  rows="3"
                  placeholder="Additional notes..."
                ></textarea>
              </div>

              <div v-if="formError" class="form-error">{{ formError }}</div>
            </form>

            <!-- View mode: display existing PO -->
            <div v-else>
              <div v-if="viewLoading" class="view-loading">Loading purchase order...</div>
              <div v-else-if="viewError" class="form-error">{{ viewError }}</div>
              <div v-else-if="existingPO" class="po-details">
                <div class="info-grid">
                  <div class="info-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ existingPO.supplier_name }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Status</div>
                    <div class="info-value">
                      <span class="badge" :class="statusBadgeClass(existingPO.status)">{{ existingPO.status }}</span>
                    </div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Quantity</div>
                    <div class="info-value">{{ existingPO.quantity }} units</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">${{ Number(existingPO.unit_cost).toFixed(2) }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Total Cost</div>
                    <div class="info-value total-cost">${{ (existingPO.quantity * existingPO.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(existingPO.expected_delivery_date) }}</div>
                  </div>
                  <div v-if="existingPO.notes" class="info-item full-span">
                    <div class="info-label">Notes</div>
                    <div class="info-value notes-value">{{ existingPO.notes }}</div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">{{ mode === 'view' ? 'Close' : 'Cancel' }}</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, watch, computed } from 'vue'
import { api } from '../api'

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
    default: 'create'
  }
})

const emit = defineEmits(['close', 'po-created'])

const form = ref({
  supplier_name: '',
  quantity: 0,
  unit_cost: null,
  expected_delivery_date: '',
  notes: ''
})

const submitting = ref(false)
const formError = ref(null)

const existingPO = ref(null)
const viewLoading = ref(false)
const viewError = ref(null)

// Pre-fill quantity when modal opens in create mode
watch(() => [props.isOpen, props.backlogItem, props.mode], ([isOpen, item, mode]) => {
  if (!isOpen || !item) return

  if (mode === 'create') {
    form.value = {
      supplier_name: '',
      // Pre-fill with shortage amount: needed minus available
      quantity: Math.max(0, item.quantity_needed - item.quantity_available),
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    }
    formError.value = null
  } else {
    // View mode: fetch the existing PO
    fetchExistingPO(item.id)
  }
}, { immediate: true })

const fetchExistingPO = async (backlogItemId) => {
  viewLoading.value = true
  viewError.value = null
  existingPO.value = null
  try {
    existingPO.value = await api.getPurchaseOrderByBacklogItem(backlogItemId)
  } catch (err) {
    viewError.value = 'Failed to load purchase order details.'
    console.error(err)
  } finally {
    viewLoading.value = false
  }
}

const submitForm = async () => {
  if (submitting.value) return
  formError.value = null
  submitting.value = true
  try {
    const payload = {
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date,
      notes: form.value.notes || undefined
    }
    const result = await api.createPurchaseOrder(payload)
    emit('po-created', result)
  } catch (err) {
    formError.value = err?.response?.data?.detail || 'Failed to create purchase order.'
    console.error(err)
  } finally {
    submitting.value = false
  }
}

const close = () => {
  emit('close')
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
}

const statusBadgeClass = (status) => {
  if (!status) return ''
  const s = status.toLowerCase()
  if (s === 'delivered' || s === 'completed') return 'success'
  if (s === 'pending' || s === 'processing') return 'warning'
  if (s === 'cancelled') return 'danger'
  return ''
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
  max-width: 600px;
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
  padding: 1.5rem 2rem;
}

.context-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
}

.context-details {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  min-width: 0;
}

.context-item-name {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.context-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.shortage-pill {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #dc2626;
  font-size: 0.813rem;
  font-weight: 600;
  padding: 0.375rem 0.75rem;
  border-radius: 20px;
  white-space: nowrap;
  flex-shrink: 0;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.optional-label {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  color: #94a3b8;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  font-family: inherit;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  color: #0f172a;
  background: white;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  font-family: inherit;
  resize: vertical;
  width: 100%;
  box-sizing: border-box;
}

.form-textarea:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #dc2626;
  font-size: 0.875rem;
  font-weight: 500;
  padding: 0.75rem 1rem;
  border-radius: 8px;
}

.view-loading {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.po-details {
  padding: 0.25rem 0;
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

.info-item.full-span {
  grid-column: 1 / -1;
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

.info-value.total-cost {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.info-value.notes-value {
  color: #475569;
  line-height: 1.5;
}

.badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.warning {
  background: #fef3c7;
  color: #92400e;
}

.badge.danger {
  background: #fecaca;
  color: #991b1b;
}

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
  background: #3b82f6;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
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

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="item-summary">
              <div class="item-summary-title">{{ backlogItem.item_name }}</div>
              <div class="item-summary-sku">SKU: {{ backlogItem.item_sku }}</div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- Create mode -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="handleSubmit">
              <div class="form-group">
                <label for="po-supplier">Supplier Name *</label>
                <input
                  id="po-supplier"
                  v-model="form.supplierName"
                  type="text"
                  class="po-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label for="po-quantity">Quantity *</label>
                  <input
                    id="po-quantity"
                    v-model.number="form.quantity"
                    type="number"
                    min="1"
                    class="po-input"
                    required
                  />
                </div>

                <div class="form-group">
                  <label for="po-unit-cost">Unit Cost *</label>
                  <input
                    id="po-unit-cost"
                    v-model.number="form.unitCost"
                    type="number"
                    min="0"
                    step="0.01"
                    class="po-input"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label for="po-delivery-date">Expected Delivery Date *</label>
                <input
                  id="po-delivery-date"
                  v-model="form.expectedDeliveryDate"
                  type="date"
                  class="po-input"
                  required
                />
              </div>

              <div class="form-group">
                <label for="po-notes">Notes</label>
                <textarea
                  id="po-notes"
                  v-model="form.notes"
                  class="po-textarea"
                  rows="3"
                  placeholder="Optional notes"
                ></textarea>
              </div>

              <div v-if="submitError" class="submit-error">{{ submitError }}</div>
            </form>

            <!-- View mode -->
            <div v-else class="po-view">
              <div v-if="viewLoading" class="po-status-message">Loading purchase order...</div>
              <div v-else-if="viewError" class="po-status-message error">{{ viewError }}</div>
              <div v-else-if="purchaseOrder" class="info-grid">
                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ purchaseOrder.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="badge" :class="statusBadgeClass(purchaseOrder.status)">
                      {{ purchaseOrder.status }}
                    </span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ purchaseOrder.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ purchaseOrder.unit_cost.toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value total-cost">${{ totalCost.toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.created_date) }}</div>
                </div>

                <div class="info-item info-item-wide" v-if="purchaseOrder.notes">
                  <div class="info-label">Notes</div>
                  <div class="info-value">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">Close</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="handleSubmit"
            >
              {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
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
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const defaultForm = () => ({
      supplierName: '',
      quantity: null,
      unitCost: null,
      expectedDeliveryDate: '',
      notes: ''
    })

    const form = ref(defaultForm())
    const submitting = ref(false)
    const submitError = ref(null)

    const purchaseOrder = ref(null)
    const viewLoading = ref(false)
    const viewError = ref(null)

    const shortage = computed(() => {
      if (!props.backlogItem) return 0
      return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
    })

    const totalCost = computed(() => {
      if (!purchaseOrder.value) return 0
      return purchaseOrder.value.quantity * purchaseOrder.value.unit_cost
    })

    const resetForm = () => {
      form.value = defaultForm()
      form.value.quantity = shortage.value > 0 ? shortage.value : null
      submitError.value = null
    }

    const loadPurchaseOrder = async () => {
      if (!props.backlogItem) return

      // Use the PO already attached by the parent if available
      if (props.backlogItem.purchase_order) {
        purchaseOrder.value = props.backlogItem.purchase_order
        viewError.value = null
        return
      }

      viewLoading.value = true
      viewError.value = null
      purchaseOrder.value = null
      try {
        purchaseOrder.value = await api.getPurchaseOrderByBacklogItem(props.backlogItem.id)
      } catch (err) {
        viewError.value = 'Failed to load purchase order'
        console.error(err)
      } finally {
        viewLoading.value = false
      }
    }

    // Reset form/view state whenever the modal opens for a (possibly new) backlog item
    watch(
      () => [props.isOpen, props.backlogItem, props.mode],
      () => {
        if (!props.isOpen) return

        if (props.mode === 'create') {
          resetForm()
        } else {
          loadPurchaseOrder()
        }
      },
      { immediate: true }
    )

    const close = () => {
      emit('close')
    }

    const handleSubmit = async () => {
      if (!props.backlogItem) return
      if (!form.value.supplierName || !form.value.quantity || !form.value.unitCost || !form.value.expectedDeliveryDate) {
        submitError.value = 'Please fill in all required fields'
        return
      }

      submitting.value = true
      submitError.value = null
      try {
        const created = await api.createPurchaseOrder({
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplierName,
          quantity: form.value.quantity,
          unit_cost: form.value.unitCost,
          expected_delivery_date: form.value.expectedDeliveryDate,
          notes: form.value.notes || null
        })
        emit('po-created', created)
        emit('close')
      } catch (err) {
        submitError.value = 'Failed to create purchase order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const statusBadgeClass = (status) => {
      const normalized = (status || '').toLowerCase()
      if (normalized === 'pending') return 'warning'
      if (normalized === 'completed' || normalized === 'received') return 'success'
      return 'default'
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    return {
      form,
      submitting,
      submitError,
      purchaseOrder,
      viewLoading,
      viewError,
      totalCost,
      close,
      handleSubmit,
      statusBadgeClass,
      formatDate
    }
  }
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
  padding: 2rem;
}

.item-summary {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-bottom: 1.5rem;
  margin-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.item-summary-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  flex: 1;
}

.item-summary-sku {
  font-size: 0.875rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-group label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #475569;
}

.po-input,
.po-textarea {
  padding: 0.625rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.938rem;
  font-family: inherit;
  color: #0f172a;
  transition: border-color 0.15s ease;
}

.po-input:focus,
.po-textarea:focus {
  outline: none;
  border-color: #64748b;
}

.po-textarea {
  resize: vertical;
}

.submit-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #991b1b;
  font-size: 0.875rem;
}

.po-status-message {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.po-status-message.error {
  color: #991b1b;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-item-wide {
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
  font-weight: 700;
  color: #0f172a;
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

.badge.warning {
  background: #fed7aa;
  color: #92400e;
}

.badge.success {
  background: #d1fae5;
  color: #065f46;
}

.badge.default {
  background: #e2e8f0;
  color: #334155;
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
  background: #0f172a;
  border: 1px solid #0f172a;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1e293b;
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

<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue';
import { useRoute } from 'vue-router';
import { shipmentsApi } from '../api/shipments';
import DataTable from '../components/common/DataTable.vue';
import StatusBadge from '../components/common/StatusBadge.vue';
import { ShipmentStatus } from '../constants/enums';
import { PERMISSIONS } from '../constants/permissions';
import { useShipmentStore } from '../stores/shipmentStore';
import { formatDate } from '../utils/format';
import type { ShipmentItem } from '../types/shipment';

const route = useRoute();
const store = useShipmentStore();
const id = String(route.params.id);
const isEditing = ref(false);
const editForm = reactive({
  estimatedArrival: '',
  remark: '',
  items: [] as Array<{ id?: string; skuId: string; skuName: string; quantity: number }>,
});

onMounted(() => store.fetchDetail(id));

function startEdit() {
  if (!store.current) return;
  editForm.estimatedArrival = store.current.estimatedArrival.slice(0, 16);
  editForm.remark = store.current.remark;
  editForm.items = store.current.items.map(item => ({
    id: item.id,
    skuId: item.skuId,
    skuName: item.skuName,
    quantity: item.quantity,
  }));
  isEditing.value = true;
}

function cancelEdit() {
  isEditing.value = false;
}

async function saveEdit() {
  try {
    await shipmentsApi.update(id, {
      estimatedArrival: new Date(editForm.estimatedArrival).toISOString(),
      remark: editForm.remark,
      items: editForm.items,
    });
    await store.fetchDetail(id);
    isEditing.value = false;
  } catch (e) {
    console.error('保存失败', e);
  }
}

function addItem() {
  editForm.items.push({ skuId: '', skuName: '', quantity: 1 });
}

function removeItem(index: number) {
  editForm.items.splice(index, 1);
}

async function receive() { await shipmentsApi.receive(id); await store.fetchDetail(id); }
async function cancel() { await shipmentsApi.cancel(id); await store.fetchDetail(id); }
</script>

<template>
  <section v-if="store.current">
    <div class="page-title">
      <h2>{{ store.current.orderNo }}</h2>
      <StatusBadge :value="store.current.status" />
    </div>
    <div class="grid two">
      <div class="panel">
        <h3>运单信息</h3>
        <template v-if="!isEditing">
          <p>承运方：{{ store.current.carrier || '-' }}</p>
          <p>追踪号：{{ store.current.trackingNo || '-' }}</p>
          <p>预计到达：{{ formatDate(store.current.estimatedArrival) }}</p>
          <p>备注：{{ store.current.remark || '-' }}</p>
          <button v-if="store.current.status === ShipmentStatus.IN_TRANSIT" class="btn" @click="receive">签收并入库</button>
          <button v-if="store.current.status === ShipmentStatus.PENDING" v-permission="PERMISSIONS.SHIPMENT_WRITE" class="btn" @click="startEdit">修改运单</button>
          <button v-if="store.current.status === ShipmentStatus.PENDING" class="btn secondary" @click="cancel">取消运单</button>
        </template>
        <template v-else>
          <div class="form-item">
            <label>预计到达</label>
            <input type="datetime-local" v-model="editForm.estimatedArrival" />
          </div>
          <div class="form-item">
            <label>备注</label>
            <textarea v-model="editForm.remark" rows="3"></textarea>
          </div>
          <div class="form-actions">
            <button class="btn" @click="saveEdit">保存</button>
            <button class="btn secondary" @click="cancelEdit">取消</button>
          </div>
        </template>
      </div>
      <div class="panel">
        <h3>物流时间线</h3>
        <p v-for="event in store.current.timeline" :key="event.id">
          <StatusBadge :value="event.status" /> {{ event.note }} · {{ event.operator }} · {{ formatDate(event.createdAt) }}
        </p>
      </div>
    </div>

    <h3>运单明细</h3>
    <template v-if="!isEditing">
      <DataTable
        :columns="[{key:'skuId',title:'SKU'},{key:'skuName',title:'名称'},{key:'quantity',title:'数量'}]"
        :data="store.current.items as any"
      />
    </template>
    <template v-else>
      <div class="panel">
        <table class="edit-table">
          <thead>
            <tr>
              <th>SKU</th>
              <th>名称</th>
              <th>数量</th>
              <th>操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(item, index) in editForm.items" :key="index">
              <td><input v-model="item.skuId" placeholder="SKU编号" /></td>
              <td><input v-model="item.skuName" placeholder="商品名称" /></td>
              <td><input type="number" v-model.number="item.quantity" min="1" /></td>
              <td><button class="btn secondary small" @click="removeItem(index)">删除</button></td>
            </tr>
          </tbody>
        </table>
        <button class="btn secondary" @click="addItem">添加商品</button>
      </div>
    </template>
  </section>
</template>

<style scoped>
.form-item { margin-bottom: 12px; }
.form-item label { display: block; margin-bottom: 4px; font-weight: 600; color: #333; }
.form-item input, .form-item textarea {
  width: 100%;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  box-sizing: border-box;
}
.form-item textarea { resize: vertical; }
.form-actions { margin-top: 16px; display: flex; gap: 8px; }
.edit-table { width: 100%; border-collapse: collapse; margin-bottom: 12px; }
.edit-table th, .edit-table td {
  padding: 8px;
  text-align: left;
  border-bottom: 1px solid #eee;
}
.edit-table input {
  width: 100%;
  padding: 6px 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  box-sizing: border-box;
}
.btn.small { padding: 4px 8px; font-size: 12px; }
</style>

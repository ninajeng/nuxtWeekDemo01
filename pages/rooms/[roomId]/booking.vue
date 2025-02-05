<script setup>
import { Icon } from '@iconify/vue';

import { bookingRoom } from '@/api/bookingRoom';

const route = useRoute();
const { roomId } = route.params;

const { formatNumber } = useFormatNumber()
const { countDay } = useDayNum()

const userStore = useUserStore();
const { userInfo } = storeToRefs(userStore);
const bookingStore = useBookingStore()
const { bookingBackup } = storeToRefs(bookingStore);

const config = useRuntimeConfig();
const token = useCookie(config.public.cookieToken).value;

const { cityZoneList, cityDefault, zoneOptions, setZones, getCityInfo } = useCityData();
setZones(cityDefault.value)
const resetZoneSelect = (city) => {
  setZones(city);
  if(zoneSelect.value) {
    zoneSelect.value.value = 0;
  }
}

// DOM
const alertDialog = ref(null)
const contactForm = ref(null)
const zoneSelect = ref(null)

const order = ref({})
const roomInfo = ref({})
const checkInDateText = ref('');
const checkOutDateText = ref('');
const daysCount = ref(0)
const discount = computed(() => daysCount.value * 1000);
const contactData = ref({
  city: cityDefault.value,
  zoneIndex: 1,
})

const phoneRegex = new RegExp('^09\\d{8}$');
const checkPhone = (phone) => phoneRegex.test(phone);

if(!bookingBackup.value.order){
  await navigateTo(`/rooms/${roomId}`);
}else{
  order.value = bookingBackup.value.order
  roomInfo.value = bookingBackup.value.roomInfo

  checkInDateText.value = order.value.checkInDate
  checkOutDateText.value = order.value.checkOutDate
  daysCount.value = countDay(order.value.checkOutDate, order.value.checkInDate)
}

const goBack = async () => {
  await navigateTo(`/rooms/${roomId}`)
}

const goRoomList = async () => {
  await navigateTo(`/rooms`)
}

const useUserInfo = () => {
  const {city, zoneIndex} = getCityInfo(userInfo.value.address.zipcode);
  const addressDetail = userInfo.value.address.detail;
  contactData.value = {...userInfo.value, city, zoneIndex, addressDetail};
  resetZoneSelect(city);
}

const isLoading = ref(false);

const confirmBooking = async () => {
  const { valid: isValid } = await contactForm.value.validate()
  if(!isValid) return

  const { name, phone, email, city, zoneIndex , addressDetail: detail} = contactData.value
  const zipcode = cityZoneList.value[city].zones[zoneIndex].zipCode;
  const data = {
    ...order.value,
    userInfo: {
      name,
      phone,
      email,
      address: {
        zipcode,
        detail
      },
    }
  }

  isLoading.value = true;
  const { status, result, message: errorMessage } = await bookingRoom(data, token)

  if(!status) {
    isLoading.value = false;
    if (errorMessage) {
      alertDialog.value.showErrorMessage({message: errorMessage})
    }else{
      alertDialog.value.showErrorMessage()
    }
    return
  }

  await navigateTo({
    name: 'booking-confirmation-bookingId',
    params: {
      bookingId: result._id
    }
  })
}

</script>

<template>
  <ClientOnly>
    <AppAlert ref="alertDialog" />
  </ClientOnly>
  <main class="pt-18 pt-md-30 bg-neutral-120">
    <section class="py-10 py-md-30 bg-primary-10">
      <div class="container">
        <button class="d-flex align-items-baseline gap-2 mb-10 bg-transparent border-0" type="button" @click="goBack">
          <Icon class="fs-5 text-neutral-100" icon="mdi:keyboard-arrow-left" />
          <h1 class="mb-0 text-neutral-100 fs-3 fw-bold">
            確認訂房資訊
          </h1>
        </button>

        <div class="row gap-10 gap-md-0">
          <div class="col-12 col-md-7">
            <section>
              <h2 class="mb-8 mb-md-10 text-neutral-100 fs-6 fs-md-4 fw-bold">
                訂房資訊
              </h2>
              <div class="d-flex flex-column gap-6">
                <div class="d-flex justify-content-between align-items-center text-neutral-100">
                  <div>
                    <h3 class="title-deco mb-2 fs-7 fw-bold">
                      選擇房型
                    </h3>
                    <p class="mb-0 fw-medium">
                      {{ roomInfo.name }}
                    </p>
                  </div>
                  <button class="bg-transparent border-0 fw-bold text-decoration-underline" type="button"
                    @click="goRoomList">
                    編輯
                  </button>
                </div>
                <div class="d-flex justify-content-between align-items-center text-neutral-100">
                  <div>
                    <h3 class="title-deco mb-2 fs-7 fw-bold">
                      訂房日期
                    </h3>
                    <p class="mb-2 fw-medium" v-dayFormat="checkInDateText">
                      入住：[day]，15:00 可入住
                    </p>
                    <p class="mb-0 fw-medium" v-dayFormat="checkOutDateText">
                      退房：[day]，12:00 前退房
                    </p>
                  </div>
                  <button class="bg-transparent border-0 fw-bold text-decoration-underline" type="button"
                    @click="goBack">
                    編輯
                  </button>
                </div>
                <div class="d-flex justify-content-between align-items-center text-neutral-100">
                  <div>
                    <h3 class="title-deco mb-2 fs-7 fw-bold">
                      房客人數
                    </h3>
                    <p class="mb-0 fw-medium">
                      {{ `${order.peopleNum} 人` }}
                    </p>
                  </div>
                  <button class="bg-transparent border-0 fw-bold text-decoration-underline" type="button"
                    @click="goBack">
                    編輯
                  </button>
                </div>
              </div>
            </section>

            <hr class="my-10 my-md-12 opacity-100 text-neutral-60">

            <section>
              <div class="d-flex justify-content-between align-items-center mb-10">
                <h2 class="mb-0 text-neutral-100 fs-6 fs-md-4 fw-bold">
                  訂房人資訊
                </h2>
                <button class="text-primary-100 bg-transparent border-0 fw-bold text-decoration-underline" type="button"
                  @click="useUserInfo">
                  套用會員資料
                </button>
              </div>

              <VeeForm v-slot="{ errors }" class="d-flex flex-column gap-6" ref="contactForm">
                <div class="text-neutral-100">
                  <label for="name" class="form-label fs-8 fs-md-7 fw-bold">姓名</label>
                  <VeeField name="name" id="name" type="text" class="form-control p-4 fs-8 fs-md-7 rounded-3 mb-1"
                    :class="{ 'is-invalid': errors.name }" rules="required|min:2" placeholder="請輸入姓名"
                    v-model="contactData.name" />
                  <VeeErrorMessage name="name">
                    <span class="text-danger">姓名需至少 2 個字</span>
                  </VeeErrorMessage>
                </div>

                <div class="text-neutral-100">
                  <label for="phone" class="form-label fs-8 fs-md-7 fw-bold">手機號碼</label>
                  <VeeField name="phone" id="phone" type="tel" class="form-control p-4 fs-8 fs-md-7 rounded-3 mb-1"
                    :class="{ 'is-invalid': errors.phone }" :rules="checkPhone" placeholder="請輸入手機號碼"
                    v-model="contactData.phone" />
                  <VeeErrorMessage name="phone">
                    <span class="text-danger">手機格式不正確</span>
                  </VeeErrorMessage>
                </div>

                <div class="text-neutral-100">
                  <label for="email" class="form-label fs-8 fs-md-7 fw-bold">電子信箱</label>
                  <VeeField name="email" id="email" type="email" class="form-control p-4 fs-8 fs-md-7 rounded-3 mb-1"
                    :class="{ 'is-invalid': errors.email }" rules="required|email" placeholder="請輸入電子信箱"
                    v-model="contactData.email" />
                  <VeeErrorMessage name="email">
                    <span class="text-danger">電子信箱格式不正確</span>
                  </VeeErrorMessage>
                </div>

                <div class="text-neutral-100">
                  <label for="address" class="form-label fs-8 fs-md-7 fw-bold">地址</label>
                  <div className="d-flex gap-2 mb-4">
                    <VeeField name="city" class="form-select w-50 p-4 text-neutral-80 fs-8 fs-md-7 fw-medium rounded-3"
                      ref="citySelect" as="select" v-model="contactData.city"
                      @change="resetZoneSelect($event.target.value)">
                      <option :value="cityName" v-for="(cityInfo, cityName) in cityZoneList" :key="`city-${cityName}`">
                        {{ cityInfo.name }}
                      </option>
                    </VeeField>
                    <VeeField name="zoneIndex"
                      class="form-select w-50 p-4 text-neutral-80 fs-8 fs-md-7 fw-medium rounded-3" ref="zoneSelect"
                      as="select" v-model="contactData.zoneIndex">
                      <option :value="index" v-for="(zone, index) in zoneOptions" :key="`zone-${index}`">
                        {{ zone }}
                      </option>
                    </VeeField>
                  </div>
                  <VeeField name="addressDetail" id="address" type="text"
                    class="form-control p-4 fs-8 fs-md-7 rounded-3 mb-1" :class="{ 'is-invalid': errors.addressDetail }"
                    placeholder="請輸入詳細地址" rules="required" v-model="contactData.addressDetail" />
                  <VeeErrorMessage name="addressDetail">
                    <span class="text-danger">地址為必填</span>
                  </VeeErrorMessage>
                </div>
              </VeeForm>
            </section>

            <hr class="my-10 my-md-12 opacity-100 text-neutral-60">

            <section>
              <h2 class="mb-8 mb-md-10 text-neutral-100 fs-6 fs-md-4 fw-bold">
                房間資訊
              </h2>
              <div class="d-flex flex-column gap-6">
                <section>
                  <h3 class="title-deco mb-4 mb-md-6 fs-7 fs-md-5 fw-bold">
                    房型基本資訊
                  </h3>
                  <ul class="d-flex gap-4 list-unstyled">
                    <li class="card-info px-4 py-5 bg-neutral-0 border border-primary-40 rounded-3">
                      <Icon class="mb-2 fs-5 text-primary-100" icon="fluent:slide-size-24-filled" />
                      <p class="mb-0 fw-bold text-neutral-80 text-nowrap">
                        {{ roomInfo.areaInfo }}
                      </p>
                    </li>
                    <li class="card-info px-4 py-5 bg-neutral-0 border border-primary-40 rounded-3">
                      <Icon class="mb-2 fs-5 text-primary-100" icon="material-symbols:king-bed" />
                      <p class="mb-0 fw-bold text-neutral-80 text-nowrap">
                        {{ roomInfo.bedInfo }}
                      </p>
                    </li>
                    <li class="card-info px-4 py-5 bg-neutral-0 border border-primary-40 rounded-3">
                      <Icon class="mb-2 fs-5 text-primary-100" icon="ic:baseline-person" />
                      <p class="mb-0 fw-bold text-neutral-80 text-nowrap">
                        {{ `2-${roomInfo.maxPeople} 人` }}
                      </p>
                    </li>
                  </ul>
                </section>

                <section>
                  <h3 class="title-deco mb-4 mb-md-6 text-neutral-100 fs-7 fs-md-5 fw-bold">
                    房間格局
                  </h3>
                  <ul class="d-flex flex-wrap gap-6 gap-md-10 p-6 fs-8 fs-md-7 bg-neutral-0 rounded-3 list-unstyled">
                    <template v-for="(layout, index) in roomInfo.layoutInfo" :key="`layout-${index}`">
                      <li class="d-flex gap-2" v-if="layout.isProvide">
                        <Icon class="fs-5 text-primary-100" icon="material-symbols:check" />
                        <p class="mb-0 text-neutral-80 fw-bold">
                          {{ layout.title }}
                        </p>
                      </li>
                    </template>
                  </ul>
                </section>

                <section>
                  <h3 class="title-deco mb-4 mb-md-6 text-neutral-100 fs-7 fs-md-5 fw-bold">
                    房內設備
                  </h3>
                  <ul
                    class="d-flex flex-wrap row-cols-2 row-cols-md-3 row-cols-lg-4 row-gap-2 column-gap-10 p-6 mb-0 fs-8 fs-md-7 bg-neutral-0 rounded-3 list-unstyled">
                    <template v-for="(facility, index) in roomInfo.facilityInfo" :key="`facility-${index}`">
                      <li class="d-flex gap-2" v-if="facility.isProvide">
                        <Icon class="fs-5 text-primary-100" icon="material-symbols:check" />
                        <p class="mb-0 text-neutral-80 fw-bold">
                          {{ facility.title }}
                        </p>
                      </li>
                    </template>

                  </ul>
                </section>

                <section>
                  <h3 class="title-deco mb-4 mb-md-6 text-neutral-100 fs-7 fs-md-5 fw-bold">
                    備品提供
                  </h3>
                  <ul
                    class="d-flex flex-wrap row-cols-2 row-cols-md-3 row-cols-lg-4 row-gap-2 p-6 mb-0 bg-neutral-0 fs-8 fs-md-7 rounded-3 list-unstyled">
                    <template v-for="(amenity, index) in roomInfo.amenityInfo" :key="`amenity-${index}`">
                      <li class="d-flex gap-2" v-if="amenity.isProvide">
                        <Icon class="fs-5 text-primary-100" icon="material-symbols:check" />
                        <p class="mb-0 text-neutral-80 fw-bold">
                          {{ amenity.title }}
                        </p>
                      </li>
                    </template>
                  </ul>
                </section>
              </div>
            </section>
          </div>

          <div class="col-12 col-md-5">
            <div
              class="confirm-form rounded-3xl d-flex flex-column gap-10 p-6 p-md-10 mx-auto ms-md-auto me-md-0 bg-neutral-0">
              <img class="img-fluid rounded-3" src="@/assets/images/room-a-1.png" alt="room-a">

              <div>
                <h2 className="mb-6 text-neutral-100 fs-6 fs-md-4 fw-bold">
                  價格詳情
                </h2>
                <div class="d-flex justify-content-between align-items-center mb-3">
                  <div class="d-flex align-items-center text-neutral-100 fw-medium">
                    <span>{{ `NT$ ${formatNumber(roomInfo.price)}` }}</span>
                    <Icon class="ms-2 me-1 text-neutral-80" icon="material-symbols:close" />
                    <span class="text-neutral-80">{{ daysCount }} 晚</span>
                  </div>
                  <span class="fw-medium">
                    {{ `NT$ ${formatNumber(roomInfo.price * daysCount)}` }}
                  </span>
                </div>
                <div class="d-flex justify-content-between align-items-center fw-medium">
                  <p class="d-flex align-items-center mb-0 text-neutral-100">
                    住宿折扣
                  </p>
                  <span class="text-primary-100">
                    {{ `-NT$ ${formatNumber(discount)}` }}
                  </span>
                </div>
                <hr class="my-6 opacity-100 text-neutral-40">
                <div class="d-flex justify-content-between align-items-center text-neutral-100 fw-bold">
                  <p class="d-flex align-items-center mb-0">
                    總價
                  </p>
                  <span>
                    {{ `NT$ ${formatNumber(roomInfo.price * daysCount - discount)}` }}
                  </span>
                </div>
              </div>

              <button class="btn btn-primary-100 py-4 text-neutral-0 fw-bold rounded-3" type="button"
                @click="confirmBooking">
                確認訂房
              </button>
            </div>
          </div>
        </div>
      </div>
    </section>

    <RoomsBookingLoading v-if="isLoading" />
  </main>
</template>

<style lang="scss" scoped>
@import "@/assets/stylesheets/page/breakpoints";

.title-deco {
  display: flex;
  align-items: center;
}

.title-deco::before {
  content: '';
  display: inline-block;
  width: 4px;
  height: 24px;
  background-color: #BF9D7D;
  border-radius: 10px;
  margin-right: 0.75rem;
}

.form-control::placeholder {
  --neutral-60: #909090;
  color: var(--neutral-60);
  font-weight: 500;
}

.card-info {
  width: 97px;
  height: 97px;
}

.flex-item {
  flex: 1 1 15%;

  @include media-breakpoint-down(md) {
    flex-basis: 40%;
  }
}

.rounded-3xl {
  border-radius: 1.25rem;
}

.confirm-form {
  position: sticky;
  top: 160px;
  max-width: 478px;

  @include media-breakpoint-down(md) {
    position: static;
    top: 0;
    max-width: auto;
  }
}
</style>
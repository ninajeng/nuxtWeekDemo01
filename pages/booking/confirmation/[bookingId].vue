<script setup>
import { Icon } from '@iconify/vue';

import { getOrderInfo } from '@/api/orderInfo';

const route = useRoute();
const { bookingId } = route.params;

const config = useRuntimeConfig();
const token = useCookie(config.public.cookieToken).value;

const userStore = useUserStore();
const { userId } = storeToRefs(userStore);

const { countDay } = useDayNum()
const { formatNumber } = useFormatNumber();

const orderInfo = ref({});
const startDate = ref('');
const endDate = ref('');
const daysCount = ref(0);
const discount = computed(() => daysCount.value * 1000);

// DOM
const alertDialog = ref(null)

const { status, result, message: errorMessage } = await getOrderInfo(bookingId, token);
if(status) {
    startDate.value = result.checkInDate.split('T')[0];
    endDate.value = result.checkOutDate.split('T')[0];
    daysCount.value = countDay(startDate.value, endDate.value)
    orderInfo.value = result
}
onMounted(()=>{
  if(!status) {
    const alertMessage = {
      message: '查無訂單' || errorMessage,
      replacePath: '/rooms'
    }
    alertDialog.value.showErrorMessage(alertMessage)
  }
})
</script>

<template>
  <AppAlert ref="alertDialog" />
  <ClientOnly>
    <main class="overflow-hidden pt-18 pt-md-30 bg-neutral-120">
      <div class="container py-10 py-md-30" v-if="orderInfo._id">
        <div class="row gap-15 gap-md-0">
          <div class="col-12 col-md-7">
            <div class="d-flex flex-column flex-md-row align-items-md-center gap-4 gap-md-10 mb-8 mb-md-10">
              <Icon class="p-2 display-4 text-neutral-0 bg-success-100 rounded-circle" icon="material-symbols:check" />
              <div class="text-neutral-0 fs-1">
                <h1 class="fw-bold">
                  {{ `恭喜，${orderInfo.userInfo.name}！` }}
                </h1>
                <p class="mb-0 fw-bold">
                  您已預訂成功
                </p>
              </div>
            </div>
            <p class="text-neutral-40 fw-medium">
              我們已發送訂房資訊及詳細內容至你的電子信箱，入住時需向櫃檯人員出示訂房人證件。
            </p>

            <hr class="my-10 my-md-20 opacity-100  text-neutral-40">

            <div class="d-flex flex-column align-items-md-start">
              <h2 class="mb-6 mb-md-10 text-neutral-0 fs-7 fs-md-5 fw-bold">
                立即查看您的訂單紀錄
              </h2>
              Nu
              <NuxtLink :to="`/user/${userId}/order`"
                class="btn btn-primary-100 px-md-15 py-4 text-neutral-0 fw-bold border-0 rounded-3">
                前往我的訂單
              </NuxtLink>
            </div>

            <hr class="my-10 my-md-20 opacity-100  text-neutral-40">

            <h2 class="mb-6 mb-md-10 text-neutral-0 fs-5 fw-bold">
              訂房人資訊
            </h2>
            <div class="d-flex flex-column gap-6">
              <div>
                <p class="mb-2 text-neutral-40 fw-medium">
                  姓名
                </p>
                <span class="text-neutral-0 fw-bold">{{ orderInfo.userInfo.name }}</span>
              </div>
              <div>
                <p class="mb-2 text-neutral-40 fw-medium">
                  手機號碼
                </p>
                <span class="text-neutral-0 fw-bold">{{ orderInfo.userInfo.phone }}</span>
              </div>
              <div>
                <p class="mb-2 text-neutral-40 fw-medium">
                  電子信箱
                </p>
                <span class="text-neutral-0 fw-bold">{{ orderInfo.userInfo.email }}</span>
              </div>
            </div>
          </div>

          <div class="col-12 col-md-5">
            <div class="rounded-3xl d-flex flex-column gap-6 gap-md-10 p-4 p-md-10 mx-auto ms-md-auto bg-neutral-0"
              style="max-width: 478px;">
              <div>
                <p class="mb-2 text-neutral-80 fs-8 fs-md-7 fw-medium">
                  預訂參考編號： {{ bookingId }}
                </p>
                <h2 class="mb-0 text-neutral-100 fs-7 fs-md-5 fw-bold">
                  即將來的行程
                </h2>
              </div>

              <img class="img-fluid rounded-3" :src="orderInfo.roomId.imageUrl" :alt="orderInfo.roomId.name">

              <section class="d-flex flex-column gap-6">
                <h3 class="d-flex align-items-center mb-6 text-neutral-80 fs-8 fs-md-6 fw-bold">
                  <p class="mb-0">
                    {{ `${orderInfo.roomId.name}，${daysCount} 晚` }}
                  </p>
                  <span class="d-inline-block mx-4 bg-neutral-80" style="width: 1px;height: 18px;" />
                  <p class="mb-0">
                    {{ `住宿人數：${orderInfo.peopleNum} 位` }}
                  </p>
                </h3>
                <div class="text-neutral-80 fs-8 fs-md-7 fw-bold">
                  <p class="title-deco mb-2" v-dayFormat="startDate">
                    入住：[day]，15:00 可入住
                  </p>
                  <p class="title-deco mb-0" v-dayFormat="endDate">
                    退房：[day]，12:00 前退房
                  </p>
                </div>

                <p class="mb-0 text-neutral-80 fs-8 fs-md-7 fw-bold">
                  {{ `NT$ ${formatNumber(orderInfo.roomId.price * daysCount - discount)}` }}
                </p>
              </section>

              <hr class="my-0 opacity-100 text-neutral-40">

              <section>
                <h3 class="title-deco mb-4 mb-md-6 text-neutral-100 fs-8 fs-md-7 fw-bold">
                  房內設備
                </h3>
                <ul
                  class="d-flex flex-wrap row-gap-2 column-gap-10 p-6 mb-0 fs-8 fs-md-7 bg-neutral-0 border border-neutral-40 rounded-3 list-unstyled">
                  <template v-for="(facility, index) in orderInfo.roomId.facilityInfo" :key="`facility-${index}`">
                    <li class="flex-item d-flex gap-2" v-if="facility.isProvide">
                      <Icon class="fs-5 text-primary-100" icon="material-symbols:check" />
                      <p class="mb-0 text-neutral-80 fw-bold">
                        {{ facility.title }}
                      </p>
                    </li>
                  </template>
                </ul>
              </section>

              <section>
                <h3 class="title-deco mb-4 mb-md-6 text-neutral-100 fs-8 fs-md-7 fw-bold">
                  備品提供
                </h3>
                <ul
                  class="d-flex flex-wrap row-gap-2 column-gap-10 p-6 mb-0 fs-8 fs-md-7 bg-neutral-0 border border-neutral-40 rounded-3 list-unstyled">
                  <template v-for="(amenity, index) in orderInfo.roomId.amenityInfo" :key="`amenity-${index}`">
                    <li class="flex-item d-flex gap-2" v-if="amenity.isProvide">
                      <Icon class="fs-5 text-primary-100" icon="material-symbols:check" />
                      <p class="mb-0 text-neutral-80 fw-bold">
                        {{ amenity.title }}
                      </p>
                    </li>
                  </template>

                </ul>
              </section>
            </div>
          </div>
        </div>
      </div>

      <picture>
        <source srcset="@/assets/images/deco-line-group-horizontal-full.svg" media="(min-width:576px)">
        <img class="w-100" src="@/assets/images/deco-line-group-horizontal-sm.svg" alt="deco-line-group">
      </picture>
    </main>
  </ClientOnly>
</template>

<style lang="scss" scoped>
@import "@/assets/stylesheets/page/breakpoints";

.rounded-3xl {
  border-radius: 1.25rem;
}

.title-deco {
  display: flex;
  align-items: center;
}

.title-deco::before {
  --color: #BF9D7D;
  content: '';
  display: inline-block;
  width: 4px;
  height: 24px;
  background-color: var(--color);
  border-radius: 10px;
  margin-right: 0.75rem;
}

.title-deco:nth-child(2)::before {
  --color: #909090;
}

.flex-item {
  flex: 1 1 26%;
  white-space: nowrap;

  @include media-breakpoint-down(md) {
    flex-basis: 40%;
  }
}
</style>
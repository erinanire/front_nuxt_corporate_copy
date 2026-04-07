<template>
  <div>
    <UiPageHeader subject="お問い合わせ" subheading="Contact" />

    <section>
      <div class="l-container--large l-container--contents">
        <template v-if="submitted">
          <p class="c-text--pre" v-html="thanksText" />
          <NuxtLink to="/" class="c-button">トップページ</NuxtLink>
        </template>
        <template v-else>
          <div class="c-form-group">
            <p
              class="c-text c-text--pre"
              v-html="response?.details?.inquiry_info"
            />
            <p class="c-text--small">
              <span class="c-form-label__required">*</span>は必須項目です。
            </p>
          </div>
          <UiAlertError
            ref="errorRef"
            v-show="errors.length > 0"
            :error="errors"
          />
          <form class="c-form">
            <div
              v-for="n in response?.details?.cols"
              :key="n.key"
              class="c-form-group"
            >
              <label :for="n.key" class="c-form-label">{{ n.title }}</label>
              <span v-if="n.required === 2" class="c-form-label__required">*</span>

              <!-- テキスト -->
              <template v-if="n.type === 1">
                <input
                  v-model="submitData[n.key]"
                  :name="n.key"
                  :id="n.key"
                  type="text"
                />
              </template>

              <!-- テキストエリア -->
              <template v-if="n.type === 2">
                <textarea
                  v-model="submitData[n.key]"
                  rows="4"
                  cols="60"
                  :name="n.key"
                  :id="n.key"
                  placeholder=""
                ></textarea>
              </template>

              <!-- ラジオボタン -->
              <template v-if="n.type === 3">
                <ul class="c-form-list">
                  <li
                    v-for="option in n.options"
                    :key="option.key"
                    class="c-form-list__item"
                  >
                    <input
                      v-model="submitData[n.key]"
                      type="radio"
                      :name="n.key"
                      :id="n.key + '-' + option.key"
                      :value="option.key"
                      class="c-form-toggle__radio"
                    />
                    <label :for="n.key + '-' + option.key">{{ option.value }}</label>
                  </li>
                </ul>
              </template>

              <!-- チェックボックス -->
              <template v-if="n.type === 4">
                <ul class="c-form-list">
                  <li
                    v-for="option in n.options"
                    :key="option.key"
                    class="c-form-list__item"
                  >
                    <input
                      v-model="submitData[n.key]"
                      type="checkbox"
                      :name="n.key"
                      :id="n.key + '-' + option.key"
                      :value="option.key"
                      class="c-form-toggle__checkbox"
                    />
                    <label :for="n.key + '-' + option.key">{{ option.value }}</label>
                  </li>
                </ul>
              </template>

              <!-- セレクトボックス -->
              <template v-if="n.type === 5">
                <select v-model="submitData[n.key]" :name="n.key" :id="n.key">
                  <option value="">選択なし</option>
                  <option
                    v-for="option in n.options"
                    :key="option.key"
                    :value="option.key"
                  >
                    {{ option.value }}
                  </option>
                </select>
              </template>

              <!-- 日付 -->
              <template v-if="n.type === 6">
                <input
                  type="date"
                  :id="n.key"
                  :name="n.key"
                  v-model="submitData[n.key]"
                  :min="getDateMin(n.attribute.arrYear)"
                  :max="getDateMax(n.attribute.arrYear)"
                />
              </template>

              <!-- ファイルアップロード -->
              <template v-if="n.type === 7">
                <input
                  type="file"
                  :name="n.key"
                  :id="n.key"
                  @change="handleFileChange($event, n.key)"
                />
              </template>

              <!-- マトリクス（チェックボックス表） -->
              <template v-if="n.type === 11">
                <table class="c-form-table">
                  <thead>
                    <tr>
                      <th></th>
                      <th v-for="col in n.attribute.col" :key="col">{{ col }}</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="(row, i_row) in n.attribute.row" :key="row">
                      <td>{{ row }}</td>
                      <td v-for="(col, i_col) in n.attribute.col" :key="col">
                        <input
                          type="checkbox"
                          :name="n.key + '_' + i_row"
                          :id="i_col"
                          v-model="submitData[n.key][i_row - 1].COL"
                          :value="{ key: i_col }"
                          class="c-form-toggle__checkbox"
                        />
                      </td>
                    </tr>
                  </tbody>
                </table>
              </template>
            </div>

            <UiSubmitButton
              :loading="loading"
              id="inquiry_item_button_confirm"
              @click.prevent="handleOnSubmit"
            >
              確認する
            </UiSubmitButton>
          </form>
        </template>
      </div>
    </section>
  </div>
</template>

<script setup>
const config = useRuntimeConfig();
const route = useRoute();

// URLパラメータから inquiry_id を取得 (/inquiry/1 → id = "1")
const inquiryId = route.params.id;

const submitted = ref(false);
const errors = ref([]);
const errorRef = ref(null);
const submitData = reactive({});
const thanksText = ref(null);
const loading = ref(false);

// inquiry_id を使って動的にフォーム定義を取得
const { data: response } = await useFetch(
  `${config.public.kurocoApiDomain}/rcms-api/1/inquiry/${inquiryId}`,
  {
    credentials: 'include',
  }
);

// フォームの初期値を設定
Object.keys(response.value.details.cols).forEach((key) => {
  const object = response.value.details.cols[key];
  if (object.type === 4) {
    // チェックボックスは配列で初期化しないと全選択バグが発生する
    submitData[object.key] = [];
  } else if (object.type === 5 || object.type === 10) {
    submitData[object.key] = '';
  } else if (object.type === 11) {
    submitData[object.key] = object.attribute.row.map(() => ({ COL: [] }));
  } else {
    submitData[object.key] = '';
  }
});

// arrYear は { 2016: 2016, ..., 2036: 2036 } 形式のオブジェクト
const getDateMin = (arrYear) => {
  const years = Object.keys(arrYear).map(Number);
  return `${Math.min(...years)}-01-01`;
};
const getDateMax = (arrYear) => {
  const years = Object.keys(arrYear).map(Number);
  return `${Math.max(...years)}-12-31`;
};

const handleFileChange = async (e, key) => {
  const fm = new FormData();
  fm.append('file', e.target.files[0]);

  try {
    const response = await $fetch(
      `${config.public.kurocoApiDomain}/rcms-api/1/upload`,
      {
        credentials: 'include',
        method: 'POST',
        body: fm,
      }
    );
    submitData[key] = response.id;
  } catch (e) {
    errors.value = e?.data?.errors || [];
  }
};

const handleOnSubmit = async () => {
  try {
    loading.value = true;
    // inquiry_id を使って動的に送信先エンドポイントを切り替える
    const response = await $fetch(
      `${config.public.kurocoApiDomain}/rcms-api/1/inquiry/${inquiryId}`,
      {
        credentials: 'include',
        method: 'POST',
        body: submitData,
      }
    );
    submitted.value = true;
    thanksText.value = response.messages?.[0];
  } catch (e) {
    errors.value = e?.data?.errors || [];
    nextTick(() => {
      errorRef.value.errorWrapperRef.scrollIntoView({
        behavior: 'smooth',
        block: 'center',
      });
    });
  } finally {
    loading.value = false;
  }
};
</script>

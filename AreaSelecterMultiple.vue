<template>
  <view class="area-selecter-multiple-2">
    <view class="box">
      <view class="left-box">
        <view @tap="handleCheckProvince(null)" :class="['province', !currentSelectedProvince.id ? 'checked' : '']">
          <text>全国</text>
          <view v-if="!selectedProvinces.length && !selectedCities.length" class="count">全</view>
        </view>
        <view
          v-for="(province, index) in provinceList"
          :key="'province-'+index"
          @tap="handleCheckProvince(province)"
          :class="['province', currentSelectedProvince.id === province.id ? 'checked' : '']">
          <text>{{ province.fullName }}</text>
          <view v-if="province.cityCount" class="count">{{ province.cityCount }}</view>
        </view>
      </view>
      <view class="right-box">
        <view v-if="currentSelectedProvince.id" @tap="handleCheckCity(null)" :class="['city', isSelectedAllCity ? 'checked' : '']">不限</view>
        <view
          v-for="(city, index) in citieslist"
          :key="'city-'+index"
          @tap="handleCheckCity(city, index)"
          :class="['city', (city.checked && !isSelectedAllCity) || (!currentSelectedProvince.id && !selectedProvinces.length && !selectedCities.length) ? 'checked' : '']">
          {{ city.fullName }}
        </view>
      </view>
    </view>
    <view class="footer">
      <nut-button
        color="#1A66FF"
        plain
        shape="square"
        @tap="handleReset"
      >
        重置
      </nut-button>
      <nut-button
        color="#1A66FF"
        shape="square"
        style="border: 1px solid #1A66FF;"
        @tap="handleSubmit"
      >
        {{ buttonText }}
      </nut-button>
    </view>
  </view>
</template>
<script>
import { districtList } from '@/apis/customForm';
import { deepClone } from '@/utils/clone';
import { defineComponent, reactive, toRefs, watch } from 'vue';

import './AreaSelecterMultiple.scss';

const currentSelectedProvince = {
  id: '',
  fullName: '',
  shortName: '',
};

export default defineComponent({
  emits: ['getAreaLists'],
  props: {
    provinces: {
      type: Array,
      default: () => ([]),
    },
    cities: {
      type: Array,
      default: () => ([]),
    },
  },
  setup (props, { emit }) {
    const state = reactive({
      a: 'area-selecter-multiple-2',
      currentSelectedProvince: deepClone(currentSelectedProvince),
      isSelectedAllCity: false,
      selectedProvinces: [], // 选择的省
      selectedCities: [], // 选择的市
      provinceList: [], // 备选列表-省,
      citieslist: [], // 备选列表-市,
      buttonText: '确定',
    });

    const handleCheckProvince = (province) => {
      if (province) {
        state.currentSelectedProvince.id = province.id;
        state.currentSelectedProvince.fullName = province.fullName;
        state.currentSelectedProvince.shortName = province.shortName;
        // 查询市
        queryDistrictList(state.currentSelectedProvince.id);
      } else {
        state.currentSelectedProvince = deepClone(currentSelectedProvince);
        state.citieslist = [
          { id: null, fullName: '全国', shortName: '全国' },
        ];
      }
    };

    const handleCheckCity = (city, index) => {
      if (!city) { // 不限
        state.isSelectedAllCity = !state.isSelectedAllCity;
        const pIndex = state.provinceList.findIndex(item => item.id === state.currentSelectedProvince.id);
        const province = state.provinceList.find(item => item.id === state.currentSelectedProvince.id);
        if (state.isSelectedAllCity) {
          state.provinceList[pIndex].cityCount = '全';
          state.selectedProvinces.push(province);
          state.citieslist.map(item => item.checked = false);
          // 把当前省下的所有市加入已选
          // const selectList = [...state.selectedCities, ...state.citieslist];
          // state.selectedCities = Array.from(new Set(selectList.map(item => item.id))).map(id => {
          //   return selectList.find(item => item.id === id);
          // });

          // 将当前省下的所有市在已选市中移除
          state.selectedCities = state.selectedCities.filter(item => item.pid !== state.currentSelectedProvince.id);
        } else {
          const selfSelectedcityLength = state.selectedCities.filter(item => item.pid === state.currentSelectedProvince.id).length;
          state.provinceList[pIndex].cityCount = selfSelectedcityLength;
          state.selectedProvinces = state.selectedProvinces.filter(item => item.id !== state.currentSelectedProvince.id);
        }
      } else if (city.id) { // 点击市
        state.isSelectedAllCity = false;
        const pIndex = state.provinceList.findIndex(item => item.id === city.pid);
        state.citieslist[index].checked = !state.citieslist[index].checked;
        if (state.citieslist[index].checked) {
          state.selectedCities.push(state.citieslist[index]);
        } else {
          state.selectedCities.splice(state.selectedCities.findIndex(item => city.id === item.id), 1);
        }
        const selfSelectedcityLength = state.selectedCities.filter(item => item.pid === city.pid).length;
        state.provinceList[pIndex].cityCount = selfSelectedcityLength;
        state.selectedProvinces = state.selectedProvinces.filter(item => item.id !== state.currentSelectedProvince.id);
      } else { // 全国
        handleReset();
      }
    };

    const showCitiesList = (list) => {
      if (list.length) {
        list.map(city => {
          return city.checked = !!state.selectedCities.find(item => item.id === city.id) || false;
        });
        const currentProvince = list[0].pid;
        const findProvince = !!state.selectedProvinces.find(item => item.id === currentProvince);
        if (findProvince) {
          state.isSelectedAllCity = true;
        } else {
          state.isSelectedAllCity = false;
        }
      }
      state.citieslist = list;
    };

    const handleReset = () => {
      state.isSelectedAllCity = true;
      state.selectedProvinces = []; // 选择的省
      state.selectedCities = []; // 选择的市
      state.currentSelectedProvince = deepClone(currentSelectedProvince);
      state.provinceList.map(item => item.cityCount = null);
      state.citieslist = [
        { id: null, fullName: '全国', shortName: '全国' },
      ];
    };

    const handleSubmit = () => {
      emit('getAreaLists', { provinces: state.selectedProvinces, cities: state.selectedCities });
    };

    const queryDistrictList = async (pid) => {
      const params = {
        pid,
        onlyFindCountryProvinceFlag: !pid,
      };
      const list = await districtList(params);
      if (!pid) {
        state.provinceList = list;
        // 父级已选
        if (props.provinces.length || props.cities.length) {
          state.selectedProvinces = props.provinces;
          state.selectedCities = props.cities;
          if (state.selectedProvinces.length) {
            state.provinceList = state.provinceList.map(item => {
              const province = state.selectedProvinces.find(province => province.id === item.id);
              return { ...item, cityCount: province ? '全' : null };
            });
          }
          if (state.selectedCities.length) {
            state.provinceList = state.provinceList.map(item => {
              const cityCount = state.selectedCities.filter(city => city.pid === item.id).length;
              return { ...item, cityCount: item.cityCount || cityCount };
            });
          }
          state.citieslist = [
            { id: null, fullName: '全国', shortName: '全国' },
          ];
        } else {
          handleReset();
        }
      } else {
        showCitiesList(list);
      }
    };

    queryDistrictList();

    watch(() => [state.selectedProvinces, state.selectedCities],
      (v) => {
        if (!v[0].length && !v[1].length) {
          state.buttonText = '确定 (已选 全国)';
        } else  {
          const txt_1 = v[0].length ? `${v[0].length}个省` : '';
          const txt_2 = v[1].length ? `${v[1].length}个市` : '';
          state.buttonText = `确定 (已选 ${txt_1} ${txt_2})`;
        }
      },
      {
        deep: true,
        immediate: true,
      },
    );

    return {
      ...toRefs(state),
      handleCheckProvince,
      handleCheckCity,
      handleReset,
      handleSubmit,
      queryDistrictList,
    };
  },
});
</script>

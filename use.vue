methods: {
  const getAreaLists = ({ provinces, cities }) => {
    if (!provinces.length && !cities.length) {
      state.showArea = '全国';

      state.form.subAreaType = 1;
      state.form.areaList = [];
    } else {
      state.showArea = [...provinces, ...cities].map(item => item.fullName).join('、');

      state.form.subAreaType = 2;
      const _provinces = (provinces || []).map(item => ({
        ...item,
        areaType: 'PROVINCE',
      }));
      const _cities = (cities || []).map(item => ({
        ...item,
        areaType: 'CITY',
      }));
      state.form.areaList = [..._provinces, ..._cities].map(item => {
        return {
          key: item.id,
          value: item.fullName,
          parentKey: item.pid,
          areaType: item.areaType,
        };
      });
    }
    state.areaVisible = false;
  };
}

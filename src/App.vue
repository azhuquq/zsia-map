<template>
  <div id="app" class="w-full h-screen relative">
    <div v-if="loading" class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 z-[1000] bg-white bg-opacity-90 px-5 py-5 rounded-md shadow-lg">
      地图加载中...
    </div>
    <div v-if="error" class="absolute top-5 left-5 right-5 bg-red-500 text-white px-3 py-2 rounded-md z-[1000]">
      {{ error }}
    </div>
    <div id="container" class="w-full h-full"></div>
  </div>
</template>

<script setup>
  import { onMounted, ref } from 'vue'
  import AMapLoader from "@amap/amap-jsapi-loader"
  import data from "./data.json"

  const loading = ref(true)
  const error = ref('')

  const loadMap = async () => {
    try {
      // 设置安全密钥
      window._AMapSecurityConfig = {
        securityJsCode: "3d58de7501eb21f183045fa6158a2565"
      }

      console.log('开始加载高德地图...')

      const AMap = await AMapLoader.load({
        key: "cef1ac2a3b954e33d37b5b19e9d4cab7",
        version: "2.0",
        plugins: ["AMap.Scale", "AMap.ToolBar", "AMap.OverView", "AMap.MapType"],
      })

      console.log('高德地图API加载成功')

      // 创建地图实例
      const map = new AMap.Map("container", {
        center: [113.5, 22.3], // 珠海市中心坐标
        zoom: 11,
        mapStyle: 'amap://styles/normal'
      })

      console.log('地图实例创建成功')

      // 添加地图控件
      map.addControl(new AMap.Scale())
      map.addControl(new AMap.ToolBar())

      // 按地址分组企业数据
      const groupedData = {}
      if (data && data.length > 0) {
        data.forEach((item) => {
          if (item.longitude && item.latitude) {
            const key = `${item.longitude},${item.latitude}`
            if (!groupedData[key]) {
              groupedData[key] = {
                position: [parseFloat(item.longitude), parseFloat(item.latitude)],
                companies: []
              }
            }
            groupedData[key].companies.push(item)
          }
        })
      }

      // 为每个位置创建标记点
      Object.values(groupedData).forEach((group) => {
        const companyCount = group.companies.length
        const isMultiple = companyCount > 1

        // 创建标记点图标
        const createIcon = (size, color, count = null) => {
          return new AMap.Icon({
            size: new AMap.Size(size, size),
            image: 'data:image/svg+xml;base64,' + btoa(`
            <svg xmlns="http://www.w3.org/2000/svg" width="${size}" height="${size}" viewBox="0 0 ${size} ${size}">
              <circle cx="${size / 2}" cy="${size / 2}" r="${size / 2 - 2}" fill="${color}" stroke="#fff" stroke-width="2"/>
              ${count ? `<text x="${size / 2}" y="${size / 2 + 4}" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold">${count}</text>` :
                `<circle cx="${size / 2}" cy="${size / 2}" r="4" fill="#fff"/>`}
            </svg>
          `),
            imageSize: new AMap.Size(size, size)
          })
        }

        // 创建标记点
        const marker = new AMap.Marker({
          position: group.position,
          title: isMultiple ? `${companyCount}家企业` : group.companies[0].企业名称,
          icon: createIcon(isMultiple ? 40 : 32, isMultiple ? '#ff6b35' : '#1890ff', isMultiple ? companyCount : null),
          anchor: 'center',
          cursor: 'pointer'
        })

        map.add(marker)

        // 创建信息窗体内容
        const createInfoContent = (companies) => {
          if (companies.length === 1) {
            const item = companies[0]
            return `
            <div style="padding: 15px; max-width: 540px; font-family: Arial, sans-serif;">
              <div style="border-bottom: 2px solid #1890ff; padding-bottom: 10px; margin-bottom: 15px;">
                <h3 style="margin: 0; color: #1890ff; font-size: 18px;">${item.企业名称}</h3>
              </div>

              <div style="margin-bottom: 12px;">
                <strong style="color: #333; display: inline-block; width: 100px;">📍 地址：</strong>
                <span style="color: #666;">${item.办公地址}</span>
              </div>

              <div style="margin-bottom: 12px;">
                <strong style="color: #333; display: inline-block; width: 100px;">👤 联系人：</strong>
                <span style="color: #666;">${item.企业联系人} ${item.联系人职务 ? ' (' + item.联系人职务 + ')' : ''}</span>
              </div>

              <div style="margin-bottom: 12px;">
                <strong style="color: #333; display: inline-block; width: 100px;">📞 电话：</strong>
                <span style="color: #666;">${item.联系电话}</span>
              </div>

              <div style="margin-bottom: 12px;">
                <strong style="color: #333; display: inline-block; width: 100px;">🏢 领域：</strong>
                <span style="color: #666;">${item.业务领域}</span>
              </div>

              <div style="margin-bottom: 12px;">
                <strong style="color: #333; display: inline-block; width: 100px;">👨‍💼 负责人：</strong>
                <span style="color: #666;">${item.企业负责人姓名} ${item.企业负责人职位 ? ' (' + item.企业负责人职位 + ')' : ''}</span>
              </div>

              ${item.公司介绍 ? `
              <div style="margin-top: 15px; padding-top: 15px; border-top: 1px solid #eee;">
                <strong style="color: #333; display: block; margin-bottom: 8px;">📋 公司介绍：</strong>
                <div style="color: #666; line-height: 1.5; max-height: 120px; overflow-y: auto;">
                  ${item.公司介绍.length > 200 ? item.公司介绍.substring(0, 200) + '...' : item.公司介绍}
                </div>
              </div>
              ` : ''}

              ${item.详情页面链接 ? `
              <div style="margin-top: 15px; text-align: center;">
                <a href="${item.详情页面链接}" target="_blank"
                   style="background: #1890ff; color: white; padding: 8px 16px;
                          text-decoration: none; border-radius: 4px; display: inline-block;">
                  查看详情
                </a>
              </div>
              ` : ''}
            </div>
          `
          } else {
            // 多个企业的情况
            return `
            <div style="padding: 15px; max-width: 600px; font-family: Arial, sans-serif;">
              <div style="border-bottom: 2px solid #ff6b35; padding-bottom: 10px; margin-bottom: 15px;">
                <h3 style="margin: 0; color: #ff6b35; font-size: 18px;">📍 此地址共有 ${companies.length} 家企业</h3>
                <p style="margin: 5px 0 0 0; color: #666; font-size: 14px;">${companies[0].办公地址}</p>
              </div>

              <div style="max-height: 400px; overflow-y: auto;">
                ${companies.map((item, index) => `
                  <div style="border: 1px solid #eee; border-radius: 8px; padding: 12px; margin-bottom: 12px; background: #fafafa;">
                    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
                      <h4 style="margin: 0; color: #1890ff; font-size: 16px;">${index + 1}. ${item.企业名称}</h4>
                      ${item.详情页面链接 ? `
                        <a href="${item.详情页面链接}" target="_blank"
                           style="background: #1890ff; color: white; padding: 4px 8px;
                                  text-decoration: none; border-radius: 3px; font-size: 12px;">
                          详情
                        </a>
                      ` : ''}
                    </div>

                    <div style="font-size: 14px; line-height: 1.4;">
                      <div style="margin-bottom: 4px;">
                        <strong>👤 联系人：</strong>${item.企业联系人} ${item.联系人职务 ? (item.联系人职务) : ''}
                      </div>
                      <div style="margin-bottom: 4px;">
                        <strong>📞 电话：</strong>${item.联系电话}
                      </div>
                      <div style="margin-bottom: 4px;">
                        <strong>🏢 领域：</strong>${item.业务领域}
                      </div>
                      <div>
                        <strong>👨‍💼 负责人：</strong>${item.企业负责人姓名} ${item.企业负责人职位 ? (item.企业负责人职位) : ''}
                      </div>
                    </div>
                  </div>
                `).join('')}
              </div>
            </div>
          `
          }
        }

        // 创建信息窗体
        const infoWindow = new AMap.InfoWindow({
          content: createInfoContent(group.companies),
          closeWhenClickMap: true,
          autoMove: true
        })

        // 点击标记点显示信息窗体
        marker.on('click', () => {
          infoWindow.open(map, marker.getPosition())
        })

        // 鼠标悬停效果
        marker.on('mouseover', () => {
          const hoverSize = isMultiple ? 44 : 36
          const hoverColor = isMultiple ? '#ff4d4f' : '#ff4d4f'
          marker.setIcon(createIcon(hoverSize, hoverColor, isMultiple ? companyCount : null))
        })

        marker.on('mouseout', () => {
          const normalSize = isMultiple ? 40 : 32
          const normalColor = isMultiple ? '#ff6b35' : '#1890ff'
          marker.setIcon(createIcon(normalSize, normalColor, isMultiple ? companyCount : null))
        })
      })

      console.log(`成功添加 ${Object.keys(groupedData).length} 个位置标记，包含 ${data.length} 家企业`)

      loading.value = false

    } catch (err) {
      console.error('地图加载失败:', err)
      error.value = `地图加载失败: ${err.message}`
      loading.value = false
    }
  }

  onMounted(() => {
    loadMap()
  })
</script>

<style scoped>
</style>

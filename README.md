import { defineConfig, loadEnv } from 'vite'
import vue from '@vitejs/plugin-vue'
import vueJsx from '@vitejs/plugin-vue-jsx'
import qiankun from 'vite-plugin-qiankun'
import { resolve } from 'path'
import type { Plugin } from 'vite'

import { viteStaticCopy } from 'vite-plugin-static-copy'
import { dynamicBase } from 'vite-plugin-dynamic-base'

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '')
  const envBase = env.VITE_APP_CONTEXT_PATH ?? '/'
  const normalizedEnvBase = envBase.endsWith('/') ? envBase : `${envBase}/`
  const dynamicBaseVar = 'window.__mapgis_dynamic_public_path__'
  const useDynamicBase = mode === 'production' || mode === 'portal.production' || mode === 'portal.production.mock'
  const placeholderBase = '/__mapgis_dynamic_public_path__/'
  const base = useDynamicBase ? placeholderBase : normalizedEnvBase
  const isPortalEnv = env.VITE_APP_CONTEXT_ENV === 'portal'
  const portalOrigin = new URL(env.VITE_APP_PORTAL_URL).origin
  const setForwardHeaders = (proxyReq, req) => {
    const host = req.headers.host
    const requestUrl = new URL(`http://${host}`)
    const forwardedHost = requestUrl.hostname
    const forwardedPort = requestUrl.port
    const prevProto = proxyReq.getHeader('X-Forwarded-Proto') as string | undefined
    const prevPort = proxyReq.getHeader('X-Forwarded-Port') as string | undefined

    proxyReq.setHeader('Host', host)
    proxyReq.setHeader('X-Forwarded-Host', forwardedHost)
    proxyReq.setHeader('X-Forwarded-Proto', prevProto ? `${prevProto},http` : 'http')
    proxyReq.setHeader('X-Forwarded-Port', prevPort ? `${prevPort},${forwardedPort}` : forwardedPort)
  }

  // 创建注入脚本的插件
  const injectBaseUrlScript = (): Plugin => {
    return {
      name: 'inject-base-url-script',
      transformIndexHtml: {
        order: 'pre', // 确保在 HTML 处理的最早阶段执行
        handler(html) {
          // 创建内联脚本，在页面加载前立即设置全局变量
          // 必须在所有模块加载之前执行，特别是 @mapgis/webclient-common
          // 直接使用配置的 base 路径，确保在开发和生产环境都能正确工作
          // normalizedBase 已经确保以 / 结尾（除非是 / 本身）
          const script = `
    <script>
      (function() {
        var fallbackBase = "${normalizedEnvBase}";

        function trimTrailingSlash(path) {
          return path.replace(/\\/+$/, '');
        }

        function normalizeBase(path) {
          if (!path || path === "/") {
            return "";
          }
          return trimTrailingSlash(path);
        }

        function runtimeBaseFrom(normalized) {
          if (!normalized) {
            return "/";
          }
          if (/^(?:[a-z]+:)?\\/\\//i.test(normalized)) {
            return normalized + "/";
          }
          return normalized + "/";
        }

        var resolvedBase = ${dynamicBaseVar} || fallbackBase;
        var normalizedBase = normalizeBase(resolvedBase);
        ${dynamicBaseVar} = normalizedBase;
        var runtimeBase = runtimeBaseFrom(normalizedBase);
        
        // 暴露运行时基础路径，便于业务代码拼接 public 资源
        window.RUNTIME_BASE_URL = runtimeBase;

        // 立即设置全局变量，必须在所有 MapGIS 库加载之前
        window.CESIUM_BASE_URL = runtimeBase + "cesiumStatic/";
        window.MAPGIS_BASE_URL = runtimeBase;
      })();
    </script>`

          // 在 <head> 标签开始后立即插入脚本（最早执行）
          return html.replace(/<head[^>]*>/, `$&${script}`)
        }
      }
    }
  }

  // qiankun 把 <script src="/__mapgis_dynamic_public_path__/assets/...">
  // 转换成了 import('/__mapgis_dynamic_public_path__/assets/...') — 字面字符串！
  // dynamicBase 不处理 HTML 中的内联脚本，导致占位符无法被动态替换
  // 这个插件在 transformIndexHtml 阶段（qiankun 之后）把字面占位符替换成变量引用
  const fixQiankunImportBase = (): Plugin => {
    return {
      name: 'fix-qiankun-import-base',
      transformIndexHtml: {
        order: 'post', // 确保在 qiankun 的 transformIndexHtml 之后执行
        handler(html) {
          const placeholder = '/__mapgis_dynamic_public_path__/'
          // 把 import('/__mapgis_dynamic_public_path__/assets/xxx.js')
          // 替换成 import(window.__mapgis_dynamic_public_path__ + '/assets/xxx.js')
          // 用全匹配避免部分替换导致破坏已有内容
          return html.replace(
            new RegExp(`import\\('${placeholder}([^']*)'\\)`, 'g'),
            (_, path) => `import(${dynamicBaseVar} + '/${path}')`
          )
        }
      }
    }
  }

  return {
    base,
    resolve: {
      alias: {
        '@': resolve(__dirname, 'src')
      }
    },
    server: {
      port: 9000,
      proxy: isPortalEnv && {
        '/portal/': {
          target: env.VITE_APP_PORTAL_URL,
          rewrite: (path: string) => path.replace(/^\/portal/, ''),
          onProxyReq: setForwardHeaders
        },
        [env.VITE_APP_PORTAL_BASE_URL]: {
          target: env.VITE_APP_PORTAL_URL,
          rewrite: (path: string) => path.replace(new RegExp(`^${env.VITE_APP_PORTAL_BASE_URL}`), ''),
          onProxyReq: setForwardHeaders
        },
        '/proxy-': {
          target: portalOrigin,
          rewrite: (path: string) => path,
          onProxyReq: setForwardHeaders
        },
        '/igs/': {
          target: portalOrigin,
          rewrite: (path: string) => path,
          onProxyReq: setForwardHeaders
        }
      },
      headers: {
        // 因为qiankun内部请求都是fetch来请求资源，所以子应用必须允许跨域
        'Access-Control-Allow-Origin': '*'
      }
    },
    css: {
      preprocessorOptions: {
        less: {
          additionalData: `@import "@/var.less";`
        }
      }
    },
    plugins: [
      vue({
        template: {
          compilerOptions: {
            // 排除 iconify 图标影子组件编译报错
            isCustomElement: tag => tag.startsWith('iconify-icon')
          }
        }
      }),
      vueJsx(),
      qiankun('dashboard', {
        // 微应用名字，与主应用注册的微应用名字保持一致
        useDevMode: true
      }),
      fixQiankunImportBase(),
      dynamicBase({
        publicPath: dynamicBaseVar,
        transformIndexHtml: true
      }),
      injectBaseUrlScript(),
      viteStaticCopy({
        targets: [
          {
            src: 'node_modules/@mapgis/cesium/Build/Cesium/*',
            dest: 'cesiumStatic'
          },
          {
            src: 'node_modules/@mapgis/webclient-cesium-plugin/dist/webclient-cesium-plugin-resource/*',
            dest: 'webclient-cesium-plugin-resource'
          },
          {
            src: 'node_modules/@mapgis/webclient-common/dist/webclient-common-resource/*',
            dest: 'webclient-common-resource'
          }
        ]
      })
    ],
    build: {
      chunkSizeWarningLimit: 4096,
      rollupOptions: {
        output: {
          manualChunks: {
            vue: ['vue', 'pinia'],
            antd: ['ant-design-vue', '@ant-design/icons-vue'],
            cesium: ['@mapgis/cesium'],
            'mapgis-common': ['@mapgis/webclient-common'],
            'mapgis-leaflet': ['@mapgis/leaflet', '@mapgis/webclient-leaflet-plugin'],
            'mapgis-mapboxgl': ['@mapgis/mapbox-gl', '@mapgis/webclient-mapboxgl-plugin'],
            'mapgis-cesium': ['@mapgis/webclient-cesium-plugin'],
            'mapgis-vue3-ui': ['@mapgis/webclient-vue3-ui'],
            'mapgis-vue3-common': ['@mapgis/webclient-vue3-common'],
            'mapgis-vue3-leaflet': ['@mapgis/webclient-vue3-leaflet'],
            'mapgis-vue3-mapboxgl': ['@mapgis/webclient-vue3-mapboxgl'],
            'mapgis-vue3-cesium': ['@mapgis/webclient-vue3-cesium']
          }
        }
      }
    }
  }
})

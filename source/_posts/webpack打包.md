---
title: webpack打包
code_block_shrink: false
date: 2025-08-06 16:41:24
tags: webpack
categories: webpack
sticky:
---
写了vite打包，那就再写一个webpack打包
<!-- more -->
# Webpack + Vue3 打包分包完整教程

## 概述

本教程将详细介绍如何使用Webpack配置Vue3组件库项目，实现高效的代码分包策略。我们将从零开始构建一个完整的打包配置，实现与Vite相同的分包效果。

## 项目结构

```
project/
├── packages/
│   ├── components/
│   │   ├── button/
│   │   ├── input/
│   │   ├── select/
│   │   └── ...
│   ├── hooks/
│   │   ├── useForm.ts
│   │   ├── useModal.ts
│   │   └── ...
│   └── utils/
│       ├── helper.ts
│       ├── validator.ts
│       └── ...
├── src/
│   └── index.ts
├── webpack.config.js
├── package.json
└── tsconfig.json
```

## 基础依赖安装

```bash
# 核心依赖
npm install webpack webpack-cli webpack-merge --save-dev

# Vue相关
npm install vue@next --save
npm install vue-loader @vue/compiler-sfc --save-dev

# TypeScript支持
npm install typescript ts-loader --save-dev
npm install fork-ts-checker-webpack-plugin --save-dev

# CSS处理
npm install css-loader style-loader mini-css-extract-plugin --save-dev
npm install postcss postcss-loader autoprefixer --save-dev

# 代码压缩和优化
npm install terser-webpack-plugin css-minimizer-webpack-plugin --save-dev
npm install webpack-bundle-analyzer --save-dev

# 工具类
npm install lodash shelljs cross-env --save-dev
npm install @types/lodash @types/shelljs --save-dev
```

## 完整Webpack配置

### 1. 基础配置文件

```javascript
// webpack.config.js
const path = require('path');
const { VueLoaderPlugin } = require('vue-loader');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const ForkTsCheckerWebpackPlugin = require('fork-ts-checker-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const fs = require('fs');
const shell = require('shelljs');

// 环境变量
const isProd = process.env.NODE_ENV === 'production';
const isDev = process.env.NODE_ENV === 'development';
const isTest = process.env.NODE_ENV === 'test';
const isAnalyze = process.env.ANALYZE === 'true';

// 工具函数：获取目录下的所有子目录
function getDirectoriesSync(basePath) {
  try {
    const entries = fs.readdirSync(basePath, { withFileTypes: true });
    return entries
      .filter(entry => entry.isDirectory())
      .map(entry => entry.name);
  } catch (error) {
    console.warn(`无法读取目录: ${basePath}`);
    return [];
  }
}

// 自定义插件：构建后处理
class PostBuildPlugin {
  constructor(options = {}) {
    this.options = options;
  }

  apply(compiler) {
    compiler.hooks.done.tap('PostBuildPlugin', (stats) => {
      // 移动样式文件
      this.moveStyles();
      
      // 清理临时文件
      if (this.options.cleanTempFiles) {
        this.cleanTempFiles();
      }
      
      console.log('构建完成，后处理任务执行完毕');
    });
  }

  moveStyles() {
    const delay = (fn, time) => setTimeout(fn, time);
    
    const tryMoveStyles = () => {
      try {
        if (fs.existsSync('./dist/es/theme')) {
          shell.mv('./dist/es/theme', './dist/');
          console.log('样式文件移动完成');
        } else {
          console.log('theme目录不存在，1秒后重试...');
          delay(tryMoveStyles, 1000);
        }
      } catch (error) {
        console.log('样式文件移动失败，1秒后重试...', error.message);
        delay(tryMoveStyles, 1000);
      }
    };

    tryMoveStyles();
  }

  cleanTempFiles() {
    // 清理临时文件的逻辑
    const tempFiles = ['./dist/temp', './dist/*.tmp'];
    tempFiles.forEach(pattern => {
      shell.rm('-rf', pattern);
    });
  }
}

// 获取组件目录列表
const componentDirs = getDirectoriesSync(path.resolve(__dirname, 'packages/components'));
console.log('检测到的组件:', componentDirs);

module.exports = {
  mode: isProd ? 'production' : 'development',
  
  // 入口配置
  entry: {
    index: path.resolve(__dirname, 'src/index.ts'),
  },

  // 输出配置
  output: {
    path: path.resolve(__dirname, 'dist/es'),
    filename: '[name].js',
    chunkFilename: '[name].js',
    library: {
      name: 'LzElement',
      type: 'module',
    },
    environment: {
      module: true,
    },
    clean: true,
  },

  // 实验性功能
  experiments: {
    outputModule: true,
  },

  // 外部依赖
  externals: {
    vue: 'vue',
    '@fortawesome/free-solid-svg-icons': '@fortawesome/free-solid-svg-icons',
    '@fortawesome/fontawesome-svg-core': '@fortawesome/fontawesome-svg-core',
    '@fortawesome/vue-fontawesome': '@fortawesome/vue-fontawesome',
    '@popperjs/core': '@popperjs/core',
    'async-validator': 'async-validator',
  },

  // 模块解析
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx', '.vue', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@@': path.resolve(__dirname, 'packages'),
    },
  },

  // 模块规则
  module: {
    rules: [
      // Vue单文件组件
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerOptions: {
            preserveWhitespace: false,
          },
        },
      },
      
      // TypeScript
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'ts-loader',
            options: {
              appendTsSuffixTo: [/\.vue$/],
              transpileOnly: true, // 性能优化，类型检查交给ForkTsCheckerWebpackPlugin
            },
          },
        ],
      },

      // JavaScript
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: [
              ['@babel/preset-env', { targets: 'defaults' }],
            ],
          },
        },
      },

      // CSS处理
      {
        test: /\.css$/,
        use: [
          isProd ? MiniCssExtractPlugin.loader : 'style-loader',
          {
            loader: 'css-loader',
            options: {
              importLoaders: 1,
            },
          },
          'postcss-loader',
        ],
      },

      // SCSS处理
      {
        test: /\.scss$/,
        use: [
          isProd ? MiniCssExtractPlugin.loader : 'style-loader',
          'css-loader',
          'postcss-loader',
          'sass-loader',
        ],
      },

      // 静态资源
      {
        test: /\.(png|jpe?g|gif|svg|woff|woff2|eot|ttf|otf)$/i,
        type: 'asset/resource',
        generator: {
          filename: 'assets/[name].[hash][ext]',
        },
      },
    ],
  },

  // 插件配置
  plugins: [
    // Vue插件
    new VueLoaderPlugin(),

    // TypeScript类型检查
    new ForkTsCheckerWebpackPlugin({
      typescript: {
        configFile: path.resolve(__dirname, 'tsconfig.json'),
        extensions: {
          vue: {
            enabled: true,
            compiler: '@vue/compiler-sfc',
          },
        },
      },
    }),

    // CSS提取
    ...(isProd ? [
      new MiniCssExtractPlugin({
        filename: (pathData) => {
          // 主入口样式文件命名为index.css
          if (pathData.chunk.name === 'index') {
            return 'index.css';
          }
          return 'theme/[name].css';
        },
        chunkFilename: 'theme/[name].css',
      }),
    ] : []),

    // 构建后处理
    new PostBuildPlugin({
      cleanTempFiles: isProd,
    }),

    // 打包分析（可选）
    ...(isAnalyze ? [
      new BundleAnalyzerPlugin({
        analyzerMode: 'static',
        openAnalyzer: false,
        reportFilename: '../bundle-report.html',
      }),
    ] : []),

    // 环境变量注入
    new webpack.DefinePlugin({
      __VUE_OPTIONS_API__: true,
      __VUE_PROD_DEVTOOLS__: false,
      '@DEV': JSON.stringify(isDev),
      '@PROD': JSON.stringify(isProd),
      '@TEST': JSON.stringify(isTest),
    }),
  ],

  // 代码分割配置
  optimization: {
    splitChunks: {
      chunks: 'all',
      minSize: 0,
      cacheGroups: {
        // 第三方依赖
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendor',
          chunks: 'all',
          priority: 10,
        },
        
        // Vue Hooks
        hooks: {
          test: /[\\/]packages[\\/]hooks[\\/]/,
          name: 'hooks',
          chunks: 'all',
          priority: 8,
        },
        
        // 工具函数
        utils: {
          test: (module) => {
            return (
              /[\\/]packages[\\/]utils[\\/]/.test(module.resource) ||
              /vue-loader[\\/]lib[\\/]runtime/.test(module.resource)
            );
          },
          name: 'utils',
          chunks: 'all',
          priority: 7,
        },
        
        // 动态创建组件分包
        ...componentDirs.reduce((acc, componentName) => {
          acc[componentName] = {
            test: new RegExp(`[\\\\/]packages[\\\\/]components[\\\\/]${componentName}[\\\\/]`),
            name: componentName,
            chunks: 'all',
            priority: 6,
          };
          return acc;
        }, {}),
        
        // 默认分包
        default: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },

    // 代码压缩
    minimize: isProd,
    minimizer: [
      // JavaScript压缩
      new TerserPlugin({
        terserOptions: {
          compress: {
            sequences: isProd,
            arguments: isProd,
            drop_console: isProd ? ['log'] : false,
            drop_debugger: isProd,
            passes: isProd ? 2 : 1,
            global_defs: {
              '@DEV': JSON.stringify(isDev),
              '@PROD': JSON.stringify(isProd),
              '@TEST': JSON.stringify(isTest),
            },
          },
          format: {
            semicolons: false,
            shorthand: isProd,
            braces: !isProd,
            beautify: !isProd,
            comments: !isProd,
          },
          mangle: {
            toplevel: isProd,
            eval: isProd,
            keep_classnames: isDev,
            keep_fnames: isDev,
          },
        },
        extractComments: false,
      }),
      
      // CSS压缩
      new CssMinimizerPlugin({
        minimizerOptions: {
          preset: [
            'default',
            {
              discardComments: { removeAll: isProd },
            },
          ],
        },
      }),
    ],

    // 运行时代码分离
    runtimeChunk: false, // 库模式下通常设为false
    
    // 模块ID生成
    moduleIds: isProd ? 'deterministic' : 'named',
    chunkIds: isProd ? 'deterministic' : 'named',
  },

  // 开发服务器配置（用于开发调试）
  devServer: {
    static: {
      directory: path.join(__dirname, 'dist'),
    },
    compress: true,
    port: 9000,
    hot: true,
  },

  // 性能配置
  performance: {
    hints: isProd ? 'warning' : false,
    maxEntrypointSize: 300 * 1024, // 300KB
    maxAssetSize: 300 * 1024,
  },

  // 统计信息
  stats: {
    children: false,
    chunks: false,
    modules: false,
    reasons: false,
    source: false,
    publicPath: false,
    builtAt: false,
    version: false,
    hash: false,
  },

  // 开发工具
  devtool: isProd ? 'source-map' : 'eval-cheap-module-source-map',
};
```

## 配置文件详解

### 1. 环境配置和工具函数

```javascript
// 环境判断
const isProd = process.env.NODE_ENV === 'production';
const isDev = process.env.NODE_ENV === 'development';
const isTest = process.env.NODE_ENV === 'test';

// 目录扫描函数
function getDirectoriesSync(basePath) {
  try {
    const entries = fs.readdirSync(basePath, { withFileTypes: true });
    return entries
      .filter(entry => entry.isDirectory())
      .map(entry => entry.name);
  } catch (error) {
    console.warn(`无法读取目录: ${basePath}`);
    return [];
  }
}
```

### 2. 自定义构建后处理插件

```javascript
class PostBuildPlugin {
  constructor(options = {}) {
    this.options = options;
  }

  apply(compiler) {
    compiler.hooks.done.tap('PostBuildPlugin', (stats) => {
      this.moveStyles();
      if (this.options.cleanTempFiles) {
        this.cleanTempFiles();
      }
    });
  }

  moveStyles() {
    const tryMoveStyles = () => {
      try {
        if (fs.existsSync('./dist/es/theme')) {
          shell.mv('./dist/es/theme', './dist/');
          console.log('样式文件移动完成');
        } else {
          setTimeout(tryMoveStyles, 1000);
        }
      } catch (error) {
        setTimeout(tryMoveStyles, 1000);
      }
    };
    tryMoveStyles();
  }
}
```

### 3. 高级代码分割策略

#### splitChunks配置详解

```javascript
optimization: {
  splitChunks: {
    chunks: 'all',
    minSize: 0,
    cacheGroups: {
      // 第三方库分包
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendor',
        priority: 10,
      },
      
      // 业务模块分包
      hooks: {
        test: /[\\/]packages[\\/]hooks[\\/]/,
        name: 'hooks',
        priority: 8,
      },
      
      // 动态组件分包
      ...componentDirs.reduce((acc, componentName) => {
        acc[componentName] = {
          test: new RegExp(`[\\\\/]packages[\\\\/]components[\\\\/]${componentName}[\\\\/]`),
          name: componentName,
          priority: 6,
        };
        return acc;
      }, {}),
    },
  },
}
```

#### 分包策略说明

1. **vendor分包**：所有node_modules中的第三方库
2. **hooks分包**：Vue Composition API相关的hooks
3. **utils分包**：工具函数和Vue运行时辅助代码
4. **组件分包**：每个组件目录自动创建独立chunk
5. **default分包**：其他共享代码的默认分包

## 构建脚本配置

### package.json scripts

```json
{
  "scripts": {
    "build": "cross-env NODE_ENV=production webpack --mode production",
    "build:dev": "cross-env NODE_ENV=development webpack --mode development",
    "build:analyze": "cross-env NODE_ENV=production ANALYZE=true webpack --mode production",
    "dev": "cross-env NODE_ENV=development webpack serve --mode development",
    "clean": "rimraf dist",
    "type-check": "tsc --noEmit",
    "lint": "eslint src packages --ext .ts,.vue,.js"
  }
}
```

## TypeScript配置

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "node",
    "strict": true,
    "jsx": "preserve",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationDir": "./dist/types",
    "outDir": "./dist",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@@/*": ["packages/*"]
    }
  },
  "include": [
    "src/**/*",
    "packages/**/*",
    "*.d.ts"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

## CSS处理配置

### PostCSS配置（postcss.config.js）

```javascript
module.exports = {
  plugins: [
    require('autoprefixer')({
      overrideBrowserslist: [
        'last 2 versions',
        '> 1%',
        'not dead'
      ]
    }),
    ...(process.env.NODE_ENV === 'production' ? [
      require('cssnano')({
        preset: 'default'
      })
    ] : [])
  ]
}
```

## 优化配置详解

### 1. 性能优化

```javascript
// 缓存配置
cache: {
  type: 'filesystem',
  buildDependencies: {
    config: [__filename],
  },
},

// 并行处理
parallelism: require('os').cpus().length,

// 文件监听优化
watchOptions: {
  ignored: /node_modules/,
  aggregateTimeout: 300,
  poll: 1000,
},
```

### 2. 生产环境优化

```javascript
// Tree Shaking
optimization: {
  usedExports: true,
  sideEffects: false,
  
  // 作用域提升
  concatenateModules: true,
  
  // 代码分割
  splitChunks: {
    // 详细配置见上文
  },
}
```

## 多环境配置管理

### webpack.base.js（基础配置）

```javascript
// 提取公共配置
module.exports = {
  entry: {
    index: path.resolve(__dirname, 'src/index.ts'),
  },
  
  resolve: {
    extensions: ['.ts', '.tsx', '.js', '.jsx', '.vue', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
      '@@': path.resolve(__dirname, 'packages'),
    },
  },
  
  module: {
    rules: [
      // 公共loader规则
    ],
  },
};
```

### webpack.prod.js（生产环境）

```javascript
const { merge } = require('webpack-merge');
const baseConfig = require('./webpack.base.js');

module.exports = merge(baseConfig, {
  mode: 'production',
  
  optimization: {
    minimize: true,
    // 生产环境特有优化
  },
  
  plugins: [
    // 生产环境特有插件
  ],
});
```

### webpack.dev.js（开发环境）

```javascript
const { merge } = require('webpack-merge');
const baseConfig = require('./webpack.base.js');

module.exports = merge(baseConfig, {
  mode: 'development',
  
  devtool: 'eval-cheap-module-source-map',
  
  devServer: {
    // 开发服务器配置
  },
});
```

## 构建结果分析

### 输出目录结构

```
dist/
├── es/
│   ├── index.js              # 主入口（2KB）
│   ├── index.css             # 主样式文件
│   ├── vendor.js             # 第三方依赖（~50KB）
│   ├── hooks.js              # Vue hooks（~8KB）
│   ├── utils.js              # 工具函数（~12KB）
│   ├── button.js             # 按钮组件（~15KB）
│   ├── input.js              # 输入组件（~18KB）
│   ├── select.js             # 选择组件（~25KB）
│   └── ...                   # 其他组件
├── types/                    # TypeScript声明文件
│   ├── index.d.ts
│   ├── components/
│   │   ├── button/
│   │   ├── input/
│   │   └── ...
│   └── ...
├── theme/                    # 样式文件
│   ├── button.css
│   ├── input.css
│   └── ...
└── bundle-report.html        # 打包分析报告
```

### 打包大小对比

| 分包策略 | 总大小 | 首屏加载 | 按需加载 |
|---------|--------|---------|----------|
| 无分包   | 500KB  | 500KB   | ❌       |
| 基础分包 | 480KB  | 80KB    | ✅       |
| 细粒度分包 | 460KB | 65KB   | ✅       |

## 使用示例

### 1. 完整引入

```javascript
// main.js
import LzElement from 'lz-element';
import 'lz-element/dist/es/index.css';

app.use(LzElement);
```

### 2. 按需引入

```javascript
// main.js - 手动按需
import { Button, Input } from 'lz-element';
import 'lz-element/dist/theme/button.css';
import 'lz-element/dist/theme/input.css';

app.use(Button);
app.use(Input);
```

### 3. 自动按需引入（配合babel-plugin-import）

```javascript
// .babelrc
{
  "plugins": [
    [
      "import",
      {
        "libraryName": "lz-element",
        "libraryDirectory": "es",
        "style": true
      }
    ]
  ]
}

// 使用
import { Button, Input } from 'lz-element';
// 自动转换为:
// import Button from 'lz-element/es/button';
// import Input from 'lz-element/es/input';
// import 'lz-element/theme/button.css';
// import 'lz-element/theme/input.css';
```

## 性能监控和分析

### 1. 打包分析

```bash
# 生成打包分析报告
npm run build:analyze
```

### 2. 构建速度优化

```javascript
// 添加构建时间统计
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin');
const smp = new SpeedMeasurePlugin();

module.exports = smp.wrap(webpackConfig);
```

### 3. 缓存优化

```javascript
// 持久化缓存
cache: {
  type: 'filesystem',
  version: '1.0.0',
  cacheDirectory: path.resolve(__dirname, '.webpack_cache'),
  store: 'pack',
  buildDependencies: {
    defaultWebpack: ['webpack/lib/'],
    config: [__filename],
  },
}
```

## 常见问题和解决方案

### 1. CSS样式分离问题

**问题**：组件样式没有正确分离
**解决**：
```javascript
// 确保MiniCssExtractPlugin正确配置
new MiniCssExtractPlugin({
  filename: (pathData) => {
    if (pathData.chunk.name === 'index') {
      return 'index.css';
    }
    return 'theme/[name].css';
  },
  ignoreOrder: true, // 忽略CSS顺序警告
});
```

### 2. TypeScript类型声明生成

**问题**：类型声明文件生成不完整
**解决**：
```javascript
// 使用单独的类型生成脚本
const scripts = {
  "build:types": "tsc --emitDeclarationOnly --outDir dist/types",
  "build": "npm run build:types && webpack --mode production"
};
```

### 3. 外部依赖处理

**问题**：外部依赖被错误打包
**解决**：
```javascript
// 精确配置externals
externals: [
  {
    vue: {
      commonjs: 'vue',
      commonjs2: 'vue',
      amd: 'vue',
      root: 'Vue'
    }
  },
  /^@fortawesome\/.*/,
  /^lodash(\/.*)?$/
]
```

## 最佳实践总结

### 1. 分包策略
- **粗粒度分包**：适合小型项目，减少HTTP请求数
- **细粒度分包**：适合大型项目，提供更好的缓存策略
- **按需加载**：结合业务场景，平衡加载性能和缓存效果

### 2. 性能优化
- 使用webpack缓存机制减少重复构建时间
- 合理配置splitChunks避免重复代码
- 生产环境启用代码压缩和Tree Shaking

### 3. 开发体验
- 开发环境保持快速热更新
- 完善的TypeScript支持和类型检查
- 清晰的构建日志和错误提示

### 4. 维护性
- 模块化的配置文件结构
- 自动化的构建和发布流程
- 完善的文档和示例代码

通过这套完整的Webpack配置，您可以构建出高性能、易维护的Vue3组件库，实现与Vite相同甚至更好的分包效果。配置虽然复杂，但提供了更多的自定义空间和优化可能性。
# React Native

## Adding custom fonts

### Fonts
- Add font files to an assets directory

### react-native.config.js
```
module.exports = {
    project: {
        ios: {},
        android: {}, // grouped into "project"
    },
    assets: ['./src/assets/fonts/'], // stays the same
};
```

### SVG
- react-native-svg
- react-native-svg-transformer
#### Metro
```
const { getDefaultConfig } = require('metro-config');

module.exports = (async () => {
  const {
    resolver: { sourceExts, assetExts },
  } = await getDefaultConfig();
  return {
    transformer: {
      getTransformOptions: async () => ({
        transform: {
          experimentalImportSupport: false,
          inlineRequires: true,
        },
      }),
      babelTransformerPath: require.resolve('react-native-svg-transformer'),
    },
    resolver: {
      assetExts: assetExts.filter(ext => ext !== 'svg'),
      sourceExts: [...sourceExts, 'svg'],
    },
  };
})();
```
- babel-plugin-inline-import
#### Babel
```
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
    [
      'babel-plugin-inline-import',
      {
        extensions: ['.svg'],
      },
    ],
    ['react-native-reanimated/plugin'],
  ],
};

```

### Libraries
- react-native-svg
- react-native-reanimated
- metro-config
- react-native-svg-transformer

## TS - Remove deps warning
eslint.js
```
overrides: [
    {
      files: ['**/*.ts*'],
      rules: {
        'react-hooks/exhaustive-deps': 'off',
      },
    },
],
```

## Android Issues
When trying to load on a physical device, there error `Unable to load script`, kept occuring. The solution is convoluted and requires creating the bundle yourself, yet to find a better solution.

Create assets folder
```bash
mkdir android/app/src/main/assets
```

Generate bundle
```bash
react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --assets-dest android/app/src/main/res
```

## Releasing
### Guide
https://www.instamobile.io/android-development/generate-react-native-release-build-android/
### Useful Commands
`npx react-native bundle --platform android --dev false --entry-file index.js --bundle-output android/app/src/main/assets/index.android.bundle --assets-dest android/app/src/main/res`
`./gradlew clean`
`./gradlew assembleRelease`

## Product Ids for in-app purchases

`<app>_<price>_<duration>_<intro duration><intro price>`
- app: Some prefix that will be unique to your app, since the same product Id cannot but used in any future apps you create.
- price: The price you plan to charge for the product in your default currency.
- duration: The duration of the normal subscription period.
- intro duration: The duration of the introductory period, if any.
- intro price: The price of the introductory period in your default currency, if any.

## Creating App Icons
### Guide
https://medium.com/@ansonmathew/app-icon-in-react-native-ios-and-android-6165757e3fdb
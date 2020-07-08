# Typescript

### Starting Project
```bash
npm init
tsc --init
npm i jest babel-jest @babel/core @babel/preset-env @babel/preset-typescript
```

### Basic tsconfig.json
```json
{
  "compilerOptions": {
    "target": "es2020",                         
    "module": "commonjs",                     
    "rootDir": "./src",
    "outDir": "./dist",                       
    "strict": true,                           
    "baseUrl": "./",                       
    "typeRoots": [
      "node_modules/@types"
    ],                       
    "types": [
      "node",
      "jest"
    ],                       
    "esModuleInterop": true, 
    "inlineSourceMap": true  
  },
  "exclude": ["tests"]
}
```

### Basic babel.config.json
```json
{
  "presets": [["@babel/preset-env", {
    "targets": {
      "node": "current"
    }}],
    "@babel/typescript"]
}
```

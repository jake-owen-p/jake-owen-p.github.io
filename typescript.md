# Typescript

### Jest & Babel
Required Packages:
```json
{"@babel/core": "^7.10.2",
"@babel/preset-env": "^7.10.2",
"@babel/preset-typescript": "^7.10.1"}
```
Basic `babel.config.json`
```json
{
  "presets": [["@babel/preset-env", {
    "targets": {
      "node": "current",
    }}],
    "@babel/typescript"]
}
```
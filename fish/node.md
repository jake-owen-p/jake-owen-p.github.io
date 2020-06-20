### TSC
```bash
set -Ua fish_user_paths /usr/local/Cellar/node/14.4.0/bin/
```

### NVM 
Add the following to `~/.config/fish/functions/nvm.fish`
```bash
function nvm
    bass source ~/.nvm/nvm.sh --no-use ';' nvm $argv
end
```

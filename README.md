# sd-rg

A wrapper around [ripgrep](https://github.com/BurntSushi/ripgrep) that mimics [sd](https://github.com/chmln/sd) for intuitive find & replace.

## Installation
```bash
curl -o ~/.local/bin/sd-rg https://raw.githubusercontent.com/elkarouh/sd-rg/main/sd-rg
chmod +x ~/.local/bin/sd-rg
```

## Usage
```bash
# Basic replacement
sd-rg 'foo' 'bar' file.txt

# Stdin
echo 'hello world' | sd-rg 'world' 'universe'

# Preview changes
sd-rg -p 'old' 'new' src/

# String literal mode (no regex)
sd-rg -F '((([])))' '' file.txt
```

Run `sd-rg --help` for full documentation.

## Dependencies

- [ripgrep](https://github.com/BurntSushi/ripgrep)

## License

MIT

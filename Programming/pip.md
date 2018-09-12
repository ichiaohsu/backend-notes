### Uninstall packages not in current requirements.txt

```bash
pip freeze | grep -v -f requirements.txt - | xargs pip uninstall -y
```
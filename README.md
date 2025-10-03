(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python3 - --uninstall

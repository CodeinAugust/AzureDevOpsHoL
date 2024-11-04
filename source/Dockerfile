FROM mcr.microsoft.com/windows/servercore:ltsc2019
RUN powershell -Command Invoke-WebRequest -Uri https://nodejs.org/dist/v14.17.0/node-v14.17.0-x64.msi -OutFile nodejs.msi; \
    Start-Process msiexec.exe -ArgumentList '/i', 'nodejs.msi', '/quiet', '/norestart' -NoNewWindow -Wait; \
    Remove-Item -Force nodejs.msi
WORKDIR /app
COPY package*.json ./
RUN powershell -Command npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]

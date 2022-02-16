![Azure AKS node.js](https://img.shields.io/badge/Azure-node.js%20AKS-blue)
# PUSH A NODE.JS APP TO AKS
## BUILD THE CONTAINER IMAGE
### SETUP NODEJS
```
sudo apt install nodejs -y
sudo apt install npm -y
npm install express-generator -g
express --view=ejs –-git demo-app
```
### CUSTOMIZE INDEX.EJS
```
  <!DOCTYPE html>
  <html>
   <head>
    <title><%= title %></title>
    <link rel='stylesheet' href='/stylesheets/style.css' />
  </head>
  <body>
    <h1><%= title %></h1>
    <p>Welcome to <%= title %></p>
    <p>This is my docker demo for nodejs</p>
    <p>Now is
    <% var d = new Date(); %>
    <%=  d.toLocaleString() %></p>
  </body>
</html>
```
## BUILD DOCKER IMAGE
### Dockerfile
```
FROM node:latest
# Create the Work Directory for app
WORKDIR /usr/src/app
# Install app dependencies
COPY package*.json ./
RUN npm install
# Bundle app source
COPY . .
EXPOSE 3000
CMD [ "npm", "start" ]
```
### .dockerignore
```
node_modules
npm-debug.log
```
## RUN COMMANDS
```
docker build . -t tpetzke/demo-app:1.0      # Build the image
docker images                               # Shows the image
docker run -p 3000:3000 -d tpetzke/demo-app # Runs the image, use -it to start interactively
```
## SETUP AKS
### 1. INSTALL Azure CLI and CREATE A RESOURCEGROUP AND REGISTRY
```
sudo apt-get install azure-cli
az login
rg=rg-aks
cr=acrthp
aks=aks-demo
az group create -l westeurope -n $rg
az acr create -g $rg -n $cr --sku Basic --admin-enabled
az acr login -n $cr
az acr credential show -n $cr
# Save the PASSWORD
```
### 2. PUSH THE IMAGE
```
docker tag tpetzke/demo-app $cr.azurecr.io/demo-app:1.0
docker push $cr.azurecr.io/demo-app:1.0
```
### 3. CREATE THE AKS CLUSTER IN THE PORTAL
```
Alternative:
az aks create -g $rg --name $aks --generate-ssh-keys
```
### 4. KUBECTL
```
az aks install-cli
az aks get-credentials -g $rg -n $aks
kubectl get nodes
```
### 5. SECRET
```
kubectl create secret docker-registry drsecret --docker-server=$cr.azurecr.io --docker-username=$cr --docker-password=<PASSWORD from above> --docker-email=tpetzke@gmx.de
kubectl describe secret
```
### 6. DEPLOYMENT DEFINITION
demo-app.yml:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: demo-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: demo-app
    spec:
      containers:
      - name: demo-app
        image: acrthp.azurecr.io/demo-app:1.0
        ports:
        - containerPort: 3000
      imagePullSecrets:
      - name: drsecret
```
### 7. DEPLOY
```
kubectl create -f demo-app.yml
```
### 8. EXPOSING THE SERVICE
```
kubectl expose deployment demo-app --type=LoadBalancer --name=demo-app
```
### 9. GET THE PUBLIC IP
```
kubectl get service/demo-app -w
```
### 10. UPDATE THE IMAGE (optional for later)
```
kubectl set image deployment/demo-app acrthp.azurecr.io/demo-app=acrthp.azurecr.io/demo
```

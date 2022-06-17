
#Deploying a Angular App to Azure Container Registry From Git
============================================================
	Reference : https://medium.com/swlh/deploy-an-angular-app-to-azure-955f0c750686

##1. Add Dockerfile with below Script Inside Near to Package.json file
	
	# Stage 1: Build
	FROM node:18-alpine AS build
	WORKDIR /usr/src/app
	COPY package.json ./
	RUN npm install
	COPY . .
	RUN npm run build

	# Stage 2: Run 
	FROM nginx:1.21.6-alpine
	# Copy compiled files from previous build stage
	COPY --from=build /usr/src/app/dist/DIMS /usr/share/nginx/html
	
##2. Add .dockerignore file with below content
	
	node_modules
	.git
	.gitignore
	
##3. Install AzureCli and Login or use Cloud Cli

##4. Create a Resource Group
	
	az group create --name {ResourceGroupName} --location eastus
	
##5. Create a Container Registry To hold docker Images

	az acr create --resource-group {ResourceGroupName} `
	--name {ContainerRegistryName} `
	--sku Basic `
	--subscription {SubcriptionName} 
	
	Example:
	
	az acr create --resource-group aztech-rg `
	--name awpindacr `
	--sku Basic `
	--subscription aztecld-westeurope-dev 
	
##6.Now, login to your new Container Registry so that we can extract some container information, such as its login server name

	6.1	
		az acr login --name {ContainerRegistryName}
		
		Ex: az acr login --name awpindacr
	
	6.2
		az acr show --name {ContainerRegistryName} `
		--query loginServer `
		--output table `
		--subscription {SubcriptionName}
		
		Ex: 
		
		az acr show --name awpindacr `
		--query loginServer `
		--output table `
		--subscription aztecld-westeurope-dev
		
		Output : awpindacr.azurecr.io
		
##7. Re-deploy your App from GitHub : your app re-build and deploy whenever you make changes to a GitHub repository

	For that Use Azure Container Tasks

##8.	Set Variables For Container Registry Tasks
	
	$ACR_NAME={ContainerRegistryName} # The name of your Azure container registry
	$GIT_USER={yourusername} # Your GitHub user account name
	$GIT_PAT=e04c06c3b571babe4e712ba0347a64119a61b61a # The PAT you generated in the previous section
	
	example :
	
	$ACR_NAME='awpindacr'       # The name of your Azure container registry`
	$GIT_USER='vaisakh-mohanan'      # Your GitHub user account name`
	$GIT_PAT='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' # The PAT you generated in the previous section
	
##9.	Create the Task
	
	az acr task create `
	--registry $ACR_NAME `
	--name {ACRTaskName} `
	--image {login server name of the container registry}/{ImageName}:{Tag} `
	--context https://github.com/$GIT_USER/Angular-Tour-of-Heroes.git#{branchName} `
	--file Dockerfile `
	--git-access-token $GIT_PAT
	
	Example1: (Not works due to github access issue)
	
	az acr task create `
	--registry $ACR_NAME `
	--name awpgitpull `
	--image awpindacr.azurecr.io/dimsui:prod `
	--context https://github.developer.allianz.io/AllianzPartnersApplicationDevelopment/$GIT_USER/DIMS_UI.git#azvm `
	--file Dockerfile `
	--git-access-token $GIT_PAT
	
	Example 2:
	
	$GIT_PAT='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'
	
	az acr task create `
	--registry $ACR_NAME `
	--name awpgitpull `
	--image awpindacr.azurecr.io/angularsample:prod `
	--context https://github.com/vaisakh-mohanan/AngularDemoApp.git#azure `
	--file Dockerfile `
	--git-access-token $GIT_PAT
	
	Example 3: with argumets
	az acr task create `
    --registry $ACR_NAME `
    --name awpgitpullnew `
    --image angularsample:$(date +%m%d%Y) `
    --arg REGISTRY_NAME=$ACR_NAME.azurecr.io `
    --context https://github.com/vaisakh-mohanan/AngularDemoApp.git `
    --file Dockerfile `
    --branch azure `
    --git-access-token $GIT_PAT
	
	
##10. Create Task from yaml file config

	10.1- Create a yaml file with build config 
	 Ex: 
		version: v1.1.0
		steps:
		  - build: -t $Registry/angular-app:$ID -t $Registry/angular-app:latest -f Dockerfile .
		  - push:
			- $Registry/angular-app:$ID
			- $Registry/angular-app:latest
			
	10.2 - Create ACR Task with yaml file reference
	
		Ex:
		  az acr task create --name awpgitpullyaml `
		  --registry $ACR_NAME `
		  --commit-trigger-enabled true `
		  --pull-request-trigger-enabled false `
		  --base-image-trigger-enabled false `
		  --context https://github.com/vaisakh-mohanan/AngularDemoApp.git#azure `
		  --assign-identity $MSI_ID `
		  --file build.yaml
		  --git-access-token $GIT_PAT

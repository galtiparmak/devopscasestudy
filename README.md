# devopscasestudy
DevOps Case Study Project
This project demonstrates a complete CI/CD pipeline using GitHub Actions, Docker, Helm, ArgoCD, and Kubernetes. The application is deployed on a Kubernetes cluster, with different environments (e.g., development and production) managed through namespaces.

For the continuous integration process, I use GitHub Actions to manage the workflow, starting with pushing changes to the repository, running unit and integration tests, building the Docker image, and finally storing the image in Docker Hub. Automated image updaters play a crucial role in ensuring that the latest images are seamlessly deployed to the target environments, reducing manual effort and minimizing the risk of human error. They also enable faster deployment of updates, including critical patches, ensuring that applications remain secure and up-to-date with minimal downtime.

I have work on two environments: development and production. The promotion strategy involves using separate GitHub branches to manage different environments, with the development branch deploying to the development namespace and the main branch deploying to the production namespace. Changes are first tested in the development environment and then promoted to production by merging them into the main branch, ensuring stability and reliability in the production environment. I chose not to use auto-merge because it is better to have a human review and verify the changes before merging them, ensuring higher accuracy and preventing potential issues from being merged automatically.

I have set necessary logic for external secret and use AWS which stores the database related values. However, I could not manage to apply those values to mysql database. Therefore, I have used hardcoded values.

I have created application helm chart which has dependency for bitnami/mysql helm chart for database configuration. Main helm chart includes one default values.yaml and two environment specific values files which are values-development and values-production. Each has their own docker tag and own database specific values like username, password, database. By database specific values, it overwrites the bitnami/mysql database configuration values.

Each environment connects to the different database endpoint. I configured my application to use placeholder values for MySQL connection details, such as username, host, port, password, and database name, in the application.properties file of my Java Spring application. These placeholders are dynamically populated at deployment time using environment variables provided by Kubernetes, ensuring flexibility and separation of configuration from the codebase. I utilized the same Docker image for all environments, differentiated only by tags like dev for development and latest for production, allowing consistent application behavior across environments without rebuilding the image.

As a result, when new changes pushed to repository from development environment, the changes go into the test, then it updates docker image for dev tag. ArgoCD applies this change by updating kubernetes pods. Same design starts for production environment in main branch when the development branch is merged to the main branch.

---Prerequisites---
Kubernetes Cluster (e.g., Minikube, Docker Desktop)
ArgoCD
Helm

---Install and run ArgoCD---
Install ArgoCD on your Kubernetes cluster:
Open cmd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Access the ArgoCD UI:
kubectl port-forward service/argocd-server -n argocd 8081:443
Open your browser and navigate to https://localhost:8080

Log in to ArgoCD:
Username: admin
Password: Fetch using kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

---Deploy the Application---
- Go to my repository(https://github.com/galtiparmak/devopscasestudy) then fork or clone my repository to your github repository.
- Create 'development' branch. Name must be the same as I provided for Github Action workflow.
- Click 'Actions' on the upperbar and enable the actions.
- Open docker hub and create image repository.
- Go to Your Github Repository->Settings->Secrets And Variables->Actions->New Repository Secret.
- Add your docker hub username as DOCKER_USERNAME and password as DOCKER_PASSWORD. Please put the same names as I provide.
- Change branch to development then go to .github->workflows for both development.yaml and production.yaml files, adjust the following line for your case image:    
  your_docker_hub_username/your_docker_hub_repo
- In development branch, go to devops-case-study-chart->values.yaml file and change image source to your case
  image:repository: your_docker_hub_username/your_docker_hub_repo
- Create pull request to merge development branch with main branch without closing development branch. Please do not close the branch.
- Wait for the github actions to complete successfully. Then, check your docker hub repository for newly created images.

Go to ArgoCD UI on your browser
- Create new app
 - Set Application Name as you wish
 - Set Project Name as default
 - Set SYNC POLICY as manual or automatic
 - Mark Autocreate Namespace box in Sync Options
 - Set Repository URL as your Github Repository URL
 - Set Revision:
    - To 'development' branch if you are creating the app for the development environment
    - To 'main' branch if you are creating the app for the production environment
 - Set Path as devops-case-study-chart
 - Set Cluster URL to your cluster url. Most probably https://kubernetes.default.svc
 - Set Namespaces
 - In the Helm section set Values Files:
    - To 'values-development.yaml' file if you are creating the app for the development environment
    - To 'values-production.yaml' file if you are creating the app for the production environment
 - Create
- Click the newly created app and synchronize it if you have selected manual synchronize

---Test the Application---
Open cmd
kubectl port-forward service/<your-app-name>-devops-case-study-chart 8080:8080 -n <your-namespace>
You may test the application on Postman with the following endpoints:
 - Create Student: POST http://localhost:8080/api/students/NAME:String/SURNAME:String/PERIOD:int
 - Get All Students: GET http://localhost:8080/api/students
 - Get Student By ID: GET http://localhost:8080/api/students/ID:Long
 - Update Student's Period By ID: PUT http://localhost:8080/api/students/ID:Long/PERIOD:int
 - Delete Student By ID: DELETE http://localhost:8080/api/students/ID:Long
 - Change Student's Name By ID: PUT http://localhost:8080/api/students/ID:Long/NAME:String
Last method is intentionally left as commanded block of code. You can remove '//' and update the github repo to test the workflow and also check
how ArgoCD updates its state.
 

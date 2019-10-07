Skaffold with `create-react-app`
===

This is a showcase for Skaffold with create-react-app.

Why?
---

If you want to develop a react app and deploy on kubernetes, you want fast feedback cycles.
Skaffold is a dedicated tool to help with this _inner dev-loop_ and it offers some nifty optimizations around script languages.
This showcase demonstrates how to get this working efficiently.

How?
---

These steps explain how this repository was created.
Use this as a guide to get started with new projects.

1. Run `create-react-app` like so:

       npx create-react-app . --typescript --use-npm

   For instructions how to work with `create-react-app` go [here](https://create-react-app.dev/docs/getting-started).

1. Add a `Dockerfile` to instruct the container builder how to construct your container:

   ```Dockerfile
   FROM node:12-alpine

   WORKDIR /app
   EXPOSE 3000
   CMD ["npm", "run", "start"]

   COPY package* ./
   RUN npm ci
   COPY . .
   ```

1. Add a `.dockerignore` file to ignore unwanted files. This is important so that Skaffold knows what files it may ignore:

   ```.dockerignore
   .git
   node_modules
   **/*.swp
   **/*.tsx~
   **/*.swn
   **/*.swo
   ```

1. Add a kubernetes manifest for your app

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: create-react-app
   spec:
     selector:
       matchLabels:
         app: create-react-app
     template:
       metadata:
         labels:
           app: create-react-app
       spec:
         containers:
         - name: create-react-app
           image: skaffold-create-react-app
           ports:
           - containerPort: 3000
   ```

1. Run `skaffold init` and add the following items:

   * Tell Skaffold to copy `.ts` or `.tsx` files into your container instead of rebuilding:

     ```yaml
     build:
       artifacts:
       - image: skaffold-create-react-app
         sync:
           infer:
           - '**/*.ts'
           - '**/*.tsx'
           - '**/*.css'
     ```

     This sync mode works entirely different to docker-compose with a local volume, as it copies the files into the running container.
     The advantage is that this will work no matter how your kubernetes cluster is set up, be it remote or local.

   * Tell Skaffold which port to forward so that you can access your app on localhost:

     ```yaml
     portForward:
     - resourceType: deployment
       resourceName: create-react-app
       port: 3000
     ```

1. Start developing with

       skaffold dev --port-forward

   This last command assumes that you have set up a kubernetes cluster. If you have not, take a look at [minikube](https://github.com/kubernetes/minikube).

1. Access your app on `http://localhost:3000`.
   When you make changes, the changed files should be sync'ed into the container and the node watcher should pick up the changes.
   In particular, the container should _not rebuild_.
   If it does nevertheless, run `skaffold dev -v debug` and look out for temporary files which should be added to `.dockerignore`.


> :warning: Note that the container runs `npm run start` which is the dev mode. When going to production, you should run `npm run build` and build a dedicated container.

**Happy hacking!**

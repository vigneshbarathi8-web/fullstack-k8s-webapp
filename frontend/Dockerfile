# Stage 1: Build - Create a build environment
# Use Node.js 20.15 image as the build stage
FROM node:20.15 AS build
# Set the working directory to /src
WORKDIR /src
# Copy package.json files to the working directory
COPY package*.json ./ 
# Install dependencies
RUN npm install
# Copy the application code to the working directory
COPY . .
# Set an environment variable
ENV VITE_PERSON_NAME=Vignesh
# Run the build command
RUN npm run build

# Stage 2: Deploy - Create a deployment environment
# Use Nginx stable-alpine image as the deployment stage
FROM nginx:stable-alpine as server
# Copy the built application from the build stage
COPY --from=build /src/dist /usr/share/nginx/html/
# Copy the Nginx configuration file
COPY ./nginx.default.conf /etc/nginx/conf.d/default.conf
# Expose port 80 for the web server
EXPOSE 80
# Run Nginx as the default command
CMD ["nginx", "-g", "daemon off;"]
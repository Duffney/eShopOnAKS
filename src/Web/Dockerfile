# Build stage
# Use a .net sdk 7.0 image as build
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
# Set the working directory to /app
WORKDIR /app
# Copy the current directory contents into the container at /app
COPY . .
# Set the working directory to /app/src/Web
WORKDIR /app/src/Web
# Restore packages
RUN dotnet restore
# Publish a release build to the /app/out directory
RUN dotnet publish -c Release -o out

# Runtime stage
# Use aspnet 7.0 as runtime image
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime

# Set the working directory to /app
WORKDIR /app
# Copy the published app from the build image
COPY --from=build /app/src/Web/out .
# Run the app
ENTRYPOINT ["dotnet", "Web.dll"]

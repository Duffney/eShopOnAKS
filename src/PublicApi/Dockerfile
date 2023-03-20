# Build stage
# Use an .NET dsk 7.0 runtime as build image
# Set the working directory to /app
# Copy the current directory contents into the container at /app
# Set the working directory to /app/src/PublicApi
# Restore packages
# Publish a release build to the /app/publish directory
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /app
COPY . .
WORKDIR /app/src/PublicApi
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish PublicApi.csproj

# Runtime stage
# Use aspnet 7.0 AS runtime
# Expose port 80
# Expose port 443
# Set the working directory to /app
# Copy the published app from the build image
# Run the application when the container launches
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime
EXPOSE 80
EXPOSE 443
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "PublicApi.dll"]
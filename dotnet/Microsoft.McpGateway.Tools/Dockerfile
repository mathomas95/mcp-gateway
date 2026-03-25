FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy solution and project files for restore
COPY Microsoft.McpGateway.sln .
COPY Microsoft.McpGateway.Management/src/Microsoft.McpGateway.Management.csproj Microsoft.McpGateway.Management/src/
COPY Microsoft.McpGateway.Tools/src/Microsoft.McpGateway.Tools.csproj Microsoft.McpGateway.Tools/src/
COPY Directory.Packages.props .

# Restore dependencies
RUN dotnet restore Microsoft.McpGateway.Tools/src/Microsoft.McpGateway.Tools.csproj -r linux-x64

# Copy source code
COPY Microsoft.McpGateway.Management/src/ Microsoft.McpGateway.Management/src/
COPY Microsoft.McpGateway.Tools/src/ Microsoft.McpGateway.Tools/src/

# Publish
RUN dotnet publish Microsoft.McpGateway.Tools/src/Microsoft.McpGateway.Tools.csproj \
    -c Release -r linux-x64 --no-restore -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8000
ENTRYPOINT ["dotnet", "Microsoft.McpGateway.Tools.dll"]

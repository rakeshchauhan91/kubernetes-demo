FROM mcr.microsoft.com/dotnet/sdk:8.0  AS build
WORKDIR /src

COPY ["MyApi/MyApi.csproj", "MyApi/"]

RUN  dotnet restore "MyApi/MyApi.csproj"

COPY . .
WORKDIR "/src/MyApi"
RUN dotnet build  MyApi.csproj -c Release -o /app/build

RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish


FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
EXPOSE 8080
EXPOSE 433
Env ASPNETCORE_URLS="http://+:8080"
ENV ASPNETCORE_ENVIRONMENT=Development 
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]

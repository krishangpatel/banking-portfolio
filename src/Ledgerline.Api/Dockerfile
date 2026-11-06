FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY Ledgerline.slnx .
COPY src/Ledgerline.Api/Ledgerline.Api.csproj src/Ledgerline.Api/
COPY src/Ledgerline.Accounts/Ledgerline.Accounts.csproj src/Ledgerline.Accounts/
COPY src/Ledgerline.Ledger/Ledgerline.Ledger.csproj src/Ledgerline.Ledger/
COPY src/Ledgerline.Transfers/Ledgerline.Transfers.csproj src/Ledgerline.Transfers/
COPY src/Ledgerline.Fraud/Ledgerline.Fraud.csproj src/Ledgerline.Fraud/
COPY src/Ledgerline.Compliance/Ledgerline.Compliance.csproj src/Ledgerline.Compliance/
COPY src/Ledgerline.Audit/Ledgerline.Audit.csproj src/Ledgerline.Audit/
COPY src/Ledgerline.Infrastructure/Ledgerline.Infrastructure.csproj src/Ledgerline.Infrastructure/
COPY tests/Ledgerline.Tests/Ledgerline.Tests.csproj tests/Ledgerline.Tests/
RUN dotnet restore src/Ledgerline.Api/Ledgerline.Api.csproj

COPY src/ src/
RUN dotnet publish src/Ledgerline.Api/Ledgerline.Api.csproj -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app
COPY --from=build /app .

EXPOSE 8080
ENTRYPOINT ["dotnet", "Ledgerline.Api.dll"]

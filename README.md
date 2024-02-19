![Logo](/images/digg.png)

# SDG Bevisdelning, inom det tekniska systemet för bevisutbyte
Här finns beskrivningen av API:et intermediationSE vilket är det API som svenska bevisproducenter implementerar, för att via Diggs förhandsgranskningstjänst, låta en användare dela bevis via det tekniska systemet för bevisutbyte (OOTS).

[Open-API-specen finns här](https://diggsweden.github.io/sdg-intermediation-se/)

För frågor om API:et, hör av er till [oss](mailto:sdg@digg.se)

## Bevisdelning och förhandsgranskning, översiktligt flöde
```mermaid
flowchart LR
    subgraph Digg
        direction LR
        FGT(Förhandsgranskningstjänsten)-->AT(SDG Auktorisationstjänst)
        AT-->LT(Legitimeringstjänst)
    end
    subgraph Bevisproducent 
        FGT-->BT(Bevistjänst)
    end
```

### Flödesbeskrivning
1. Förhandsgranskningstjänsten återautentiserar andvändaren mha SDG auktorisationstjänst som i sin tur nyttjar legitimeringstjänsten Sweden Connect.
2. En lyckat autentisering ger förhandsgranskningstjänsten ett åtkomstintyg (accesstoken) som skickas med till bevisproducentens bevistjänst. 
3. Med hjälp av åtkomstintyget auktoriserar bevisproducenten API-anropet och lämnar ett bevissvar till förhandsgranskningstjänsten där användaren avgör om beviset ska delas och föras över för vidare användning i sitt pågående förfarande. 

## Bevisdelning och förhandsgranskning, detaljerat flöde
```mermaid
sequenceDiagram
    participant USR as User
    participant PDFE as ProcedureDemoFE
    participant PSFE as PreviewSpaceFE
    participant PSBE as PreviewSpaceBE
    participant IMSE as IntermediationSE
    participant RED as Redis
    box Auktorisation
    participant AS as AuthorizationServer
    participant ATS as AuthenticationService
    end
    participant EP as EvidenceProvider

    USR ->> PDFE: 1. Start procedure
    PDFE ->> PSFE: 2. Redirect to preview
    PSFE ->> PSBE: 3. GET /authorize/login

    PSBE ->> AS: 4. AuthenticationRequest, with redirect uri, to AuthorizationServer
    Note over AS,ATS: 5. User authentication
    AS ->> PSBE: 6. Response from AuthorizationServer to PSBE including authentication code
    PSBE ->> AS: 7. IdTokenRequest(code)
    AS ->> PSBE: 8. TokenResponse(idtoken, accesstoken, refreshtoken)
    PSBE ->> RED: 9. Store accesstoken for reuse
    PSBE ->> AS: 10. AuthorizationRequest, with redirect uri, to AuthorizationServer
    Note over AS,ATS: 11. Evidence provider authorization
    AS ->> PSBE: 12. Redirect from AuthorizationServer to PSBE including authorization code
    PSBE ->> AS: 13. AccessTokenRequest(code)
    AS ->> PSBE: 14. TokenResponse(accesstoken, refreshtoken)
    PSBE ->> RED: 15. Store accesstoken for reuse
    PSBE ->> PSFE: 16. Redirect from PSBE to PSFE
    PSFE ->> PSBE: 17. Post /documents/available to fetch evidences
    PSBE ->> AS: 18. Get user info (accesstoken)
    AS ->> PSBE: 19. User info response
    PSBE ->> PSBE: 20. Match evidence request against user information
    PSBE ->> IMSE: 21. POST /evidence-files (accesstoken)
    IMSE ->> EP: 22. GET evidence files from provider (accesstoken)
    EP ->> EP: 23. Validate accesstoken
    EP -->> IMSE: 24. Return documents
    IMSE -->> PSBE: 26. Return documents
    PSBE -->> PSFE: 27. Show documents
    PSFE -->> PDFE: 28. Share documents
```

### Flödesbeskrivning
1. Användaren besöker förfarandet och förbereder en bevisbegäran
2. Användaren initierar autentisering
3. Användaren väljer att logga in och initierar legitimeringsförfrågan
4. PreviewSpaceBE bygger ihop och skickar ett authentication request till AuthorizationServer, samtidigt blir användaren omdirigerad till id-tjänsten
5. Användaren legitimerar sig i id-tjänsten
6. AuthorizationServer svarar förbestämd callback endpoint med en authentication code
7. PreviewSpaceBE bygger ihop och skickar ett idtoken request till AuthorizationServer
8. Authorization svarar med idtoken, accesstoken, refreshtoken.
9. PreviewSpaceBE sparar ner accesstoken i redis.
10. PreviewSpaceBE bygger ihop och skickar ett authorization request till AuthorizationServer
11. Auktorisering av klient
12. AuthorizationServer svarar förbestämd callback endpoint med en authorization code
13. PreviewSpaceBE bygger ihop och skickar ett accesstoken request
14. AuthorizationServer svarar med accesstoken och refreshtoken
15. PreviewSpaceBE sparar ner accesstoken i redis
16. Användaren blir omdirigerad tillbaka till PSFE
17. Bevishämtning initieras
18. PreviewSpaceBE skickar en förfrågan till AuthorizationServer för att hämta användarinformation
19. AuthorizationServer svarar med användarinformation
20. PreviewSpaceBE matchar information från evidence request mot användarinformationen.
21. PreviewSpaceBE skickar förfrågan, som innehåller accesstoken, för bevishämtning till IntermediationSE
22. IntermediationSE skickar förfrågan, som innehåller accesstoken, till bevisproducent
23. Bevisproducent validerar accesstoken
24. Bevisproducent returnerar IntermediationSE med bevis
25. IntermediationSE returnerar bevis till PreviewSpaceBE
26. PreviewSpaceBE returnerar bevis till PreviewSpaceFE som visar upp hämtade bevis för användaren.
27. Användare väljer att dela bevis med förfarande


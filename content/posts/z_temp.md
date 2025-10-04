Initializing with Entra authority: https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0
https://login.microsoftonline.com:443 "GET /44db8ac1-2253-4cb0-8bb5-0252a10a64f0/v2.0/.well-known/openid-configuration HTTP/1.1" 200 1805

okay it basically get the OpenID config from here: https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/v2.0/.well-known/openid-configuration 
this config: 
OpenID Configuration (formatted):
```json
{
    "token_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/token",
    "token_endpoint_auth_methods_supported": [
        "client_secret_post",
        "private_key_jwt",
        "client_secret_basic"
    ],
    "jwks_uri": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/discovery/v2.0/keys",
    "response_modes_supported": [
        "query",
        "fragment",
        "form_post"
    ],
    "subject_types_supported": [
        "pairwise"
    ],
    "id_token_signing_alg_values_supported": [
        "RS256"
    ],
    "code_challenge_methods_supported": [
        "plain",
        "S256"
    ],
    "response_types_supported": [
        "code",
        "id_token",
        "code id_token",
        "id_token token"
    ],
    "scopes_supported": [
        "openid",
        "profile",
        "email",
        "offline_access"
    ],
    "issuer": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/v2.0",
    "request_uri_parameter_supported": false,
    "userinfo_endpoint": "https://graph.microsoft.com/oidc/userinfo",
    "authorization_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/authorize",
    "device_authorization_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/devicecode",
    "http_logout_supported": true,
    "frontchannel_logout_supported": true,
    "end_session_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/logout",
    "claims_supported": [
        "sub",
        "iss",
        "cloud_instance_name",
        "cloud_instance_host_name",
        "cloud_graph_host_name",
        "msgraph_host",
        "aud",
        "exp",
        "iat",
        "auth_time",
        "acr",
        "nonce",
        "preferred_username",
        "name",
        "tid",
        "ver",
        "at_hash",
        "c_hash",
        "email"
    ],
    "kerberos_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/kerberos",
    "tenant_region_scope": "EU",
    "cloud_instance_name": "microsoftonline.com",
    "cloud_graph_host_name": "graph.windows.net",
    "msgraph_host": "graph.microsoft.com",
    "rbac_url": "https://pas.windows.net"
}
```
anschliessend der authorize link 
https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/authorize?client_id=86cf2f4d-d1f3-4af4-9be2-fb4c089cffd5&response_type=code&redirect_uri=http%3A%2F%2Flocalhost%3A5001%2Fcallback&scope=User.Read+offline_access+openid+profile&state=8f9a6415-6c46-48e1-bdb1-800232615e1c&code_challenge=BRlCLPwUybzs-0zo0fVJsXDpe_DuncvadxcocS7mKA0&code_challenge_method=S256&nonce=3d179adcb2c3756ffec18375f8a4df7bb0608d14432a5c8760d0684450bf3631&client_info=1&sso_reload=true

dananch aufruf des token endpoints: 
DEBUG:urllib3.connectionpool:https://login.microsoftonline.com:443 "POST /44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/token HTTP/1.1" 200 6780
DEBUG:msal.token_cache:event={
    "client_id": "86cf2f4d-d1f3-4af4-9be2-fb4c089cffd5",
    "data": {
        "claims": null,
        "client_id": "86cf2f4d-d1f3-4af4-9be2-fb4c089cffd5",
        "code": "1.AS8AwYrbRFMisEyLtQJSoQpk8E0vz4bz0fRKm-L7TAic_9WwAAAvAA.AgABBAIAAABlMNzVhAPUTrARzfQjWPtKAwDs_wUA9P_--Pf26h-xPui6Lv77FPga0DnpDjIun34YFZkbSwgtgZiWNFHPtY7Admy57HNTWeo4OAE1w3RWN_zBA8bo0f71ium3GZ-hMFbwkJL2nA-LvicaoO8qA523Nv9pHXESVooC94hfBSxiqBaONjR0moL8GIvxfaAkgUsNDm-NAvliCvx3x3ksDHUgCGoA0rl0AP_9zDGZJypSDNZ42IcWgcgAuFamn92p5BwbiZ2M9JcNClynAsc0cS_rQn6w_TTtH1QZ2GRjoVN2n241qY1vuQhA2V3v7_lAggsvI50I3NU7rzK3zOLg1FG_czeWT7blaD_z-MpMI6-5eykH508ZVCzZ_JaYJHSLyi4mvQYWrBJ277oJEu6s8aDd1BC_pvCelOFDkH5u9-Tny5vmK506vhZkrSGE7qbwfXdNP3flus4JIXVVQu0MfubHQymVtOA1w3wPAxHVs3quGNDZ5JRMMQ6UlAndpy2mJtblEiwHoqUJS-t5xM4gCO49BoSf2FucrjMdeTTqOuFc2CtBpwOHx0KLJj_wuWv3UAItHcc_usi03QwPjqyg-Jf-0QcjjfWxwlYJ4x8xZHNTye7e2egNl1X2_I1O0hAkakp9IhVviL8Jc1VXRD9ibU-6VFRfSWvNVvocQF4CugGGqhvVzPzIS0QfKXMyinywS00B4SRQrfe3jJkF1GkP46di3L4pGAjxEv02Jnen1PHmY9oIi42VN_IfPPfjVutWpg1WNUTdV66hX8_wPH2bhLGeAUkywdFePskJFhijXzYprFF87j0LeRXcm6s4_wB86mdp3rDdJdXTDNrZWYM8n5GYiEzHRScYaQD3R-qyXXdNHBCH2IQQAChdbE_ocrDYl6mrBDmF-70-3IlbAZSXYK7JSK6PGCHm_a3XnNG4ZjMfebbNOQjxgvQx3FkdOQ6f76tNdeC398mFK04qyOzFauE72h7C632561t5HSf2q3jblH5eS61-s24edXB1y79v9FV7sKWvaEngoYBBzQlt0dQh7wyeLDxGXY_r7OCmiCJA8LUkbGYj9ZSBPG8kn20eks9gDhG1_utqhUo3wvQdWMRL5MZkvgC4XTkYe05Zb-ZWCwOPZ-BKygDD40wcZJMvGWdl3ZbVrNj_FlF_r1lP5Yew3xubIerFgjQQOmg_GexEWcOTmaLUH0Uxl-O2SVTNBCG2RkNcDRL74KV6EKN6iROqxD2soV7l85x43VXdl9yRUDl7MBu97oV98-1jsdfyWdN1DsRSaweOKic7Ss9aLH-EwffGUIipR-MING4bLavKsALy49lkdjj-8gdvn12HluZ7We8C2Awa_XlSjD03_FWVOwBxOmCypUmG526o8OeMseGCPe6cNQRy8GhOC7t2cDpYzHjxsJGZMEmdWKUPrPGJBFd9PMYUeugmVcRgW0kqdR9qz0KeFblHslz-rXbkeqHlHOHkvAqa-_8x_nqpx_f1CrgiieJzc0Y8PcJVDG792A8XL93x775U1sAhxZ0eP5YuPJncv_cClu-kg0qFVfdC5SD8OVHj6w",
        "code_verifier": "269il5sYz.DPVoLpdhyvRQuMmZnbgOFASqKf40Ca1rE",
        "redirect_uri": "http://localhost:5001/callback",
        "scope": [
            "User.Read",
            "openid",
            "profile",
            "offline_access"
        ]
    },
    "environment": "login.microsoftonline.com",
    "grant_type": "authorization_code",
    "params": null,
    "response": {
        "access_token": "********",
        "client_info": "eyJ1aWQiOiIwMDAwMDAwMC0wMDAwLTAwMDAtOWM1Yi1lNTk1ZDFkMjllOTkiLCJ1dGlkIjoiOTE4ODA0MGQtNmM2Ny00YzViLWIxMTItMzZhMzA0YjY2ZGFkIn0",
        "expires_in": 4968,
        "ext_expires_in": 4968,
        "id_token": "********",
        "refresh_token": "********",
        "scope": "User.Read profile openid email",
        "token_type": "Bearer"
    },
    "scope": [
        "User.Read",
        "profile",
        "openid",
        "email"
    ],
    "token_endpoint": "https://login.microsoftonline.com/44db8ac1-2253-4cb0-8bb5-0252a10a64f0/oauth2/v2.0/token"
}

anschliessend noch: 
graph.microsoft.com:443
DEBUG:urllib3.connectionpool:https://graph.microsoft.com:443 "GET /v1.0/me HTTP/1.1" 200 None
INFO:werkzeug:127.0.0.1 - - [03/Oct/2025 14:55:26] "GET /callback?code=1.AS8AwYrbRFMisEyLtQJSoQpk8E0vz4bz0fRKm-L7TAic_9WwAAAvAA.AgABBAIAAABlMNzVhAPUTrARzfQjWPtKAwDs_wUA9P_--Pf26h-xPui6Lv77FPga0DnpDjIun34YFZkbSwgtgZiWNFHPtY7Admy57HNTWeo4OAE1w3RWN_zBA8bo0f71ium3GZ-hMFbwkJL2nA-LvicaoO8qA523Nv9pHXESVooC94hfBSxiqBaONjR0moL8GIvxfaAkgUsNDm-NAvliCvx3x3ksDHUgCGoA0rl0AP_9zDGZJypSDNZ42IcWgcgAuFamn92p5BwbiZ2M9JcNClynAsc0cS_rQn6w_TTtH1QZ2GRjoVN2n241qY1vuQhA2V3v7_lAggsvI50I3NU7rzK3zOLg1FG_czeWT7blaD_z-MpMI6-5eykH508ZVCzZ_JaYJHSLyi4mvQYWrBJ277oJEu6s8aDd1BC_pvCelOFDkH5u9-Tny5vmK506vhZkrSGE7qbwfXdNP3flus4JIXVVQu0MfubHQymVtOA1w3wPAxHVs3quGNDZ5JRMMQ6UlAndpy2mJtblEiwHoqUJS-t5xM4gCO49BoSf2FucrjMdeTTqOuFc2CtBpwOHx0KLJj_wuWv3UAItHcc_usi03QwPjqyg-Jf-0QcjjfWxwlYJ4x8xZHNTye7e2egNl1X2_I1O0hAkakp9IhVviL8Jc1VXRD9ibU-6VFRfSWvNVvocQF4CugGGqhvVzPzIS0QfKXMyinywS00B4SRQrfe3jJkF1GkP46di3L4pGAjxEv02Jnen1PHmY9oIi42VN_IfPPfjVutWpg1WNUTdV66hX8_wPH2bhLGeAUkywdFePskJFhijXzYprFF87j0LeRXcm6s4_wB86mdp3rDdJdXTDNrZWYM8n5GYiEzHRScYaQD3R-qyXXdNHBCH2IQQAChdbE_ocrDYl6mrBDmF-70-3IlbAZSXYK7JSK6PGCHm_a3XnNG4ZjMfebbNOQjxgvQx3FkdOQ6f76tNdeC398mFK04qyOzFauE72h7C632561t5HSf2q3jblH5eS61-s24edXB1y79v9FV7sKWvaEngoYBBzQlt0dQh7wyeLDxGXY_r7OCmiCJA8LUkbGYj9ZSBPG8kn20eks9gDhG1_utqhUo3wvQdWMRL5MZkvgC4XTkYe05Zb-ZWCwOPZ-BKygDD40wcZJMvGWdl3ZbVrNj_FlF_r1lP5Yew3xubIerFgjQQOmg_GexEWcOTmaLUH0Uxl-O2SVTNBCG2RkNcDRL74KV6EKN6iROqxD2soV7l85x43VXdl9yRUDl7MBu97oV98-1jsdfyWdN1DsRSaweOKic7Ss9aLH-EwffGUIipR-MING4bLavKsALy49lkdjj-8gdvn12HluZ7We8C2Awa_XlSjD03_FWVOwBxOmCypUmG526o8OeMseGCPe6cNQRy8GhOC7t2cDpYzHjxsJGZMEmdWKUPrPGJBFd9PMYUeugmVcRgW0kqdR9qz0KeFblHslz-rXbkeqHlHOHkvAqa-_8x_nqpx_f1CrgiieJzc0Y8PcJVDG792A8XL93x775U1sAhxZ0eP5YuPJncv_cClu-kg0qFVfdC5SD8OVHj6w&client_info=eyJ1aWQiOiIwMDAwMDAwMC0wMDAwLTAwMDAtOWM1Yi1lNTk1ZDFkMjllOTkiLCJ1dGlkIjoiOTE4ODA0MGQtNmM2Ny00YzViLWIxMTItMzZhMzA0YjY2ZGFkIn0&state=8f9a6415-6c46-48e1-bdb1-800232615e1c&session_state=00991399-c6a8-283c-2bb9-19ee780b2546 HTTP/1.1" 302 -
INFO:werkzeug:127.0.0.1 - - [03/Oct/2025 14:55:33] "GET / HTTP/1.1" 200 -


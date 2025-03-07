<script>
    import ExpandContainer from "$lib/ExpandContainer.svelte";
    import * as yup from "yup";
    import {extractFormErrors} from "../../../utils/helpers.js";
    import {onMount} from "svelte";
    import Button from "$lib/Button.svelte";
    import {postProviderLookup, postProvider} from "../../../utils/dataFetchingAdmin.js";
    import Input from "$lib/inputs/Input.svelte";
    import Switch from "$lib/Switch.svelte";
    import CheckIcon from "$lib/CheckIcon.svelte";
    import OptionSelect from "$lib/OptionSelect.svelte";
    import PasswordInput from "$lib/inputs/PasswordInput.svelte";
    import {
        REGEX_CLIENT_NAME,
        REGEX_PEM, REGEX_PROVIDER_SCOPE,
        REGEX_URI
    } from "../../../utils/constants.js";
    import JsonPathDesc from "./JsonPathDesc.svelte";
    import Textarea from "$lib/inputs/Textarea.svelte";

    export let idx = -1;
    export let onSave;

    const inputWidth = '25rem';

    let expandContainer = false;
    let isLoading = false;
    let err = '';
    let lookupSuccess = false;
    let success = false;
    let timer;

    let showRootPem = false;

    let configLookup = {
        issuer: null,
        metadata_url: null,
        danger_allow_insecure: false,
        root_pem: null,
    };
    let config = {
        enabled: true,
        typ: 'oidc',

        // fixed values after lookup
        issuer: '',
        danger_allow_insecure: false,
        authorization_endpoint: '',
        token_endpoint: '',
        token_auth_method_basic: false,
        userinfo_endpoint: '',
        use_pkce: true,
        client_secret_basic: true,
        client_secret_post: false,

        // user defined values
        name: '',
        client_id: '',
        client_secret: '',
        scope: '',
        root_pem: null,

        admin_claim_path: null,
        admin_claim_value: null,
        mfa_claim_path: null,
        mfa_claim_value: null,
        // maybe additional ones in the future like client_logo
    }
    // TODO add "the big ones" as templates in the future
    let modes = ['OIDC', 'Auto', 'Custom', 'Github', 'Google'];
    let mode = modes[0];
    $: isAuto = mode === modes[1];
    $: isCustom = mode === modes[2];
    $: isOidc = mode === modes[0];
    $: isSpecial = !isAuto && !isCustom && !isOidc;

    let formErrors = {};
    const schemaConfig = yup.object().shape({
        issuer: yup.string().trim().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128").required('Required'),
        authorization_endpoint: yup.string().url().required('Required'),
        token_endpoint: yup.string().url().required('Required'),
        userinfo_endpoint: yup.string().url().required('Required'),

        name: yup.string().trim().matches(REGEX_CLIENT_NAME, "Can only contain: 'a-zA-Z0-9À-ÿ- ', length max: 128").required('Required'),
        client_id: yup.string().trim().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128").required('Required'),
        client_secret: yup.string().trim().max(256, "Max 256 characters"),
        scope: yup.string().trim().matches(REGEX_PROVIDER_SCOPE, "Can only contain: 'a-zA-Z0-9-_/ ', length max: 128").required('Required'),
        root_pem: yup.string().trim().nullable().matches(REGEX_PEM, "Invalid PEM certificate"),

        admin_claim_path: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
        admin_claim_value: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
        mfa_claim_path: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
        mfa_claim_value: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
    });
    const schemaLookup = yup.object().shape({
        issuer: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
        metadata_url: yup.string().trim().nullable().matches(REGEX_URI, "Can only contain URI safe characters, length max: 128"),
        root_pem: yup.string().trim().nullable().matches(REGEX_PEM, "Valid PEM certificate"),
    });

    // hook for templated values
    $: if (mode) {
        // reset values
        lookupSuccess = false;
        formErrors = {};
        configLookup = {
            issuer: null,
            metadata_url: null,
            danger_allow_insecure: false,
            root_pem: null,
        };

        switch (mode) {
            case 'Github':
                // Github does not implement metadata lookup -> configure manually
                config = {
                    enabled: true,

                    // fixed values after lookup
                    issuer: 'github.com',
                    danger_allow_insecure: false,
                    authorization_endpoint: 'https://github.com/login/oauth/authorize',
                    token_endpoint: 'https://github.com/login/oauth/access_token',
                    token_auth_method_basic: false,
                    userinfo_endpoint: 'https://api.github.com/user',
                    use_pkce: false,
                    client_secret_basic: true,
                    client_secret_post: true,

                    // user defined values
                    name: 'Github',
                    client_id: '',
                    client_secret: '',
                    scope: 'read:user',
                    root_pem: null,

                    admin_claim_path: null,
                    admin_claim_value: null,
                    mfa_claim_path: '$.two_factor_authentication',
                    mfa_claim_value: 'true',
                    // maybe additional ones in the future like client_logo
                };
                break;
            case 'Google':
                // Google supports oidc metadata lookup
                configLookup = {
                    issuer: 'accounts.google.com',
                    metadata_url: null,
                    danger_allow_insecure: false,
                    root_pem: null,
                }
                onSubmitLookup();
                break;
            default:
                config = {
                    enabled: true,
                    typ: 'oidc',

                    // fixed values after lookup
                    issuer: '',
                    danger_allow_insecure: false,
                    authorization_endpoint: '',
                    token_endpoint: '',
                    token_auth_method_basic: false,
                    userinfo_endpoint: '',
                    use_pkce: true,
                    client_secret_basic: true,
                    client_secret_post: true,

                    // user defined values
                    name: '',
                    client_id: '',
                    client_secret: '',
                    scope: '',
                    root_pem: null,

                    admin_claim_path: null,
                    admin_claim_value: null,
                    mfa_claim_path: null,
                    mfa_claim_value: null,
                    // maybe additional ones in the future like client_logo
                };
        }
    }

    $: if (success) {
        timer = setTimeout(() => {
            onSave();
            success = false;
            expandContainer = false;
            resetValues();
        }, 1500);
    }

    onMount(() => {
        return () => {
            expandContainer = false;
            clearTimeout(timer);
        }
    });

    async function onSubmitConfig() {
        const valid = await validateFormConfig();
        if (!valid) {
            return;
        }

        if (!config.use_pkce && !config.client_secret) {
            err = 'Must at least be a confidential client or use PKCE';
            return;
        }

        err = '';
        isLoading = true;

        if (config.root_pem) {
            // make sure we reset to false, which is what a user would expect
            config.danger_allow_insecure = false;
            config.root_pem = config.root_pem.trim();
        }

        if (isAuto) {
            config.typ = 'custom';
        } else {
            config.typ = mode.toLowerCase();
        }
        config.scope = config.scope.trim();

        let res = await postProvider(config);
        if (res.ok) {
            success = true;
        } else {
            let body = await res.json();
            if (body.message.includes('InvalidCertificate')) {
                err = 'Insecure connection not allowed';
            } else {
                err = body.message;
            }
        }

        isLoading = false;
    }

    async function onSubmitLookup() {
        const valid = await validateFormLookup();
        if (!valid) {
            err = 'Invalid input';
            return;
        }

        err = '';
        isLoading = true;

        let res = await postProviderLookup(configLookup);
        if (res.ok) {
            const body = await res.json();
            config.issuer = body.issuer;
            config.authorization_endpoint = body.authorization_endpoint;
            config.danger_allow_insecure = body.danger_allow_insecure;
            config.token_endpoint = body.token_endpoint;
            config.userinfo_endpoint = body.userinfo_endpoint;
            config.token_auth_method_basic = body.token_auth_method_basic;
            config.use_pkce = body.use_pkce;
            config.client_secret_basic = body.client_secret_basic;
            // we want to enable basic only if it is supported for better compatibility out of the box
            config.client_secret_post = !body.client_secret_basic && body.client_secret_post;
            config.scope = body.scope;
            config.root_pem = body.root_pem;

            lookupSuccess = true;
        } else {
            let body = await res.json();
            if (body.message.includes('InvalidCertificate')) {
                err = 'Insecure connection not allowed';
            } else {
                err = body.message;
            }
        }

        isLoading = false;
    }

    function resetValues() {
        configLookup = {
            issuer: null,
            metadata_url: null,
            danger_allow_insecure: false,
            root_pem: null,
        };
        config = {
            enabled: true,
            issuer: '',
            danger_allow_insecure: false,
            authorization_endpoint: '',
            token_endpoint: '',
            userinfo_endpoint: '',
            use_pkce: true,
            client_secret_basic: true,
            client_secret_post: false,
            scope: '',
            root_pem: null,
            admin_claim_path: null,
            admin_claim_value: null,
            mfa_claim_path: null,
            mfa_claim_value: null,
        }
        lookupSuccess = false;
        showRootPem = false;
    }

    async function validateFormConfig() {
        formErrors = {};
        try {
            await schemaConfig.validate(config, {abortEarly: false});

            if (config.client_secret && !(config.client_secret_basic || config.client_secret_post)) {
                err = 'You have given a client secret, but no client auth method is active';
                return false;
            } else {
                err = 'Invalid input';
            }

            return true;
        } catch (err) {
            formErrors = extractFormErrors(err);
            err = 'Invalid input';
            return false;
        }
    }

    async function validateFormLookup() {
        formErrors = {};
        try {
            await schemaLookup.validate(configLookup, {abortEarly: false});
            if (!configLookup.issuer && !configLookup.metadata_url) {
                formErrors.issuer = 'Required';
                formErrors.metadata_url = formErrors.issuer;
                return false;
            }

            return true;
        } catch (err) {
            formErrors = extractFormErrors(err);
            return false;
        }
    }

</script>

<ExpandContainer bind:idx bind:show={expandContainer}>
    <div class="header font-label" slot="header">
        ADD NEW AUTHENTICATION PROVIDER
    </div>

    <div class="container" slot="body">
        <div class="header">
            Type
        </div>
        <div class="ml mb">
            <OptionSelect bind:value={mode} options={modes}/>
        </div>

        {#if !lookupSuccess}
            <div class="header">
                Custom Root CA PEM
            </div>
            <div class="ml mb">
                <Switch bind:selected={showRootPem}/>
            </div>

            {#if showRootPem}
                <Textarea
                        rows={17}
                        name="rootPem"
                        placeholder="-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----"
                        bind:value={configLookup.root_pem}
                        bind:error={formErrors.root_pem}
                >
                    Root Certificate in PEM format
                </Textarea>
            {:else}
                <div class="header">
                    Allow insecure TLS certificates
                </div>
                <div class="ml mb">
                    <Switch bind:selected={configLookup.danger_allow_insecure}/>
                </div>
            {/if}
        {/if}

        {#if isOidc && !lookupSuccess}
            <Input
                    type="url"
                    name="issuer"
                    bind:value={configLookup.issuer}
                    bind:error={formErrors.issuer}
                    placeholder="Issuer URL"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    on:enter={onSubmitLookup}
            >
                ISSUER URL
            </Input>

            <Button on:click={onSubmitLookup} bind:isLoading level={1} width="6rem">
                LOOKUP
            </Button>
        {:else if isAuto && !lookupSuccess}
            <Input
                    type="url"
                    name="metadata"
                    bind:value={configLookup.metadata_url}
                    bind:error={formErrors.metadata_url}
                    placeholder=".../.well-known/openid-configuration"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    on:enter={onSubmitLookup}
            >
                METADATA URL
            </Input>

            <Button on:click={onSubmitLookup} bind:isLoading level={1} width="6rem">
                LOOKUP
            </Button>
        {:else if isSpecial || isCustom || lookupSuccess}
            {#if showRootPem}
                <Textarea
                        rows={17}
                        name="rootPem"
                        placeholder="-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----"
                        bind:value={config.root_pem}
                        bind:error={formErrors.root_pem}
                        disabled={lookupSuccess}
                >
                    Root Certificate in PEM format
                </Textarea>
            {:else}
                <div class="header">
                    Allow insecure TLS certificates
                </div>
                <div class="ml mb">
                    {#if lookupSuccess}
                        <CheckIcon bind:check={config.danger_allow_insecure}/>
                    {:else}
                        <Switch bind:selected={config.danger_allow_insecure}/>
                    {/if}
                </div>
            {/if}

            <Input
                    type="url"
                    name="issuer"
                    bind:value={config.issuer}
                    bind:error={formErrors.issuer}
                    placeholder="Issuer URL"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    disabled={lookupSuccess}
            >
                ISSUER URL
            </Input>

            <Input
                    type="url"
                    name="auth_endpoint"
                    bind:value={config.authorization_endpoint}
                    bind:error={formErrors.authorization_endpoint}
                    placeholder="Authorization Endpoint"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    disabled={lookupSuccess}
            >
                AUTHORIZATION ENDPOINT
            </Input>

            <Input
                    type="url"
                    name="token_endpoint"
                    bind:value={config.token_endpoint}
                    bind:error={formErrors.token_endpoint}
                    placeholder="Token Endpoint"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    disabled={lookupSuccess}
            >
                TOKEN ENDPOINT
            </Input>

            <Input
                    type="url"
                    name="userinfo_endpoint"
                    bind:value={config.userinfo_endpoint}
                    bind:error={formErrors.userinfo_endpoint}
                    placeholder="Userinfo Endpoint"
                    on:input={validateFormLookup}
                    width={inputWidth}
                    disabled={lookupSuccess}
            >
                USERINFO ENDPOINT
            </Input>

            <div class="header">
                Use PKCE
            </div>
            <div class="ml">
                {#if lookupSuccess}
                    <CheckIcon bind:check={config.use_pkce}/>
                {:else}
                    <Switch bind:selected={config.use_pkce}/>
                {/if}
            </div>

            <div class="desc">
                The scope the client should use when redirecting to the login.<br>
                Provide the values separated by space.
            </div>
            <Input
                    name="scope"
                    bind:value={config.scope}
                    bind:error={formErrors.scope}
                    placeholder="openid profile email"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                SCOPE
            </Input>

            <div class="desc">
                Client name for the Rauthy login form
            </div>
            <Input
                    name="client_name"
                    bind:value={config.name}
                    bind:error={formErrors.name}
                    placeholder="Client Name"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                CLIENT NAME
            </Input>

            <div class="desc">
                Client ID given by the auth provider
            </div>
            <Input
                    name="client_id"
                    bind:value={config.client_id}
                    bind:error={formErrors.client_id}
                    autocomplete="off"
                    placeholder="Client ID"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                CLIENT ID
            </Input>

            <div class="desc">
                Client Secret given by the auth provider.<br>
                At least a client secret or PKCE is required.
            </div>
            <PasswordInput
                    name="client_secret"
                    bind:value={config.client_secret}
                    bind:error={formErrors.client_secret}
                    autocomplete="off"
                    placeholder="Client Secret"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                CLIENT SECRET
            </PasswordInput>

            <div class="desc">
                <p>
                    The authentication method to use on the <code>/token</code> endpoint.<br>
                    Most providers should work with <code>basic</code>, some only with <code>post</code>.
                    In rare situations, you need both, while it can lead to errors with others.
                </p>
            </div>
            <div class="switchRow">
                <div>
                    client_secret_basic
                </div>
                <Switch
                        bind:selected={config.client_secret_basic}
                />
            </div>
            <div class="switchRow">
                <div>
                    client_secret_post
                </div>
                <Switch
                        bind:selected={config.client_secret_post}
                />
            </div>

            <JsonPathDesc/>
            <div class="desc">
                <p>
                    You can map a user to be a rauthy admin depending on an upstream ID claim.
                </p>
            </div>
            <Input
                    name="admin_claim_path"
                    bind:value={config.admin_claim_path}
                    bind:error={formErrors.admin_claim_path}
                    placeholder="$.roles.*"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                ADMIN CLAIM PATH
            </Input>
            <Input
                    name="admin_claim_value"
                    bind:value={config.admin_claim_value}
                    bind:error={formErrors.admin_claim_value}
                    placeholder="rauthy_admin"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                ADMIN CLAIM VALUE
            </Input>

            <div class="desc">
                <p>
                    If your provider issues a claim indicating that the user has used at least 2FA during
                    login, you can specify the mfa claim path.
                </p>
            </div>
            <Input
                    name="mfa_claim_path"
                    bind:value={config.mfa_claim_path}
                    bind:error={formErrors.mfa_claim_path}
                    placeholder="$.amr.*"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                MFA CLAIM PATH
            </Input>
            <Input
                    name="mfa_claim_value"
                    bind:value={config.mfa_claim_value}
                    bind:error={formErrors.mfa_claim_value}
                    placeholder="mfa"
                    on:input={validateFormConfig}
                    width={inputWidth}
            >
                MFA CLAIM VALUE
            </Input>

            <Button on:click={onSubmitConfig} bind:isLoading level={1} width="6rem">
                SAVE
            </Button>

            <Button on:click={resetValues} bind:isLoading level={4} width="6rem">
                RESET
            </Button>
        {/if}

        {#if success}
            <div class="success">
                Success
            </div>
        {/if}

        {#if err}
            <div class="err">
                {err}
            </div>
        {/if}
    </div>
</ExpandContainer>

<style>
    .container {
        padding: 10px;
    }

    .desc {
        display: flex;
        flex-direction: column;
        margin: .5rem;
    }

    .desc > p {
        margin: .2rem 0;
    }

    .err {
        color: var(--col-err);
    }

    .err, .success {
        margin: 0 7px;
    }

    .header {
        display: flex;
        font-size: .9rem;
        margin-left: 10px;
    }

    .ml {
        margin-left: .5rem;
    }

    .mb {
        margin-bottom: .5rem;
    }

    .success {
        color: var(--col-ok);
    }

    .switchRow {
        margin-bottom: .25rem;
        padding-left: .5rem;
        display: grid;
        grid-template-columns: 9rem 1fr;
    }
</style>

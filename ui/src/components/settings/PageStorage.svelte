<script>
    import { slide } from "svelte/transition";
    import ApiClient from "@/utils/ApiClient";
    import CommonHelper from "@/utils/CommonHelper";
    import { pageTitle } from "@/stores/app";
    import { setErrors } from "@/stores/errors";
    import { addSuccessToast } from "@/stores/toasts";
    import tooltip from "@/actions/tooltip";
    import PageWrapper from "@/components/base/PageWrapper.svelte";
    import Field from "@/components/base/Field.svelte";
    import RedactedPasswordInput from "@/components/base/RedactedPasswordInput.svelte";
    import SettingsSidebar from "@/components/settings/SettingsSidebar.svelte";

    $pageTitle = "Files storage";

    let originalFormSettings = {};
    let formSettings = {};
    let isLoading = false;
    let isSaving = false;

    $: initialHash = JSON.stringify(originalFormSettings);

    $: hasChanges = initialHash != JSON.stringify(formSettings);

    loadSettings();

    async function loadSettings() {
        isLoading = true;

        try {
            const settings = (await ApiClient.settings.getAll()) || {};
            init(settings);
        } catch (err) {
            ApiClient.errorResponseHandler(err);
        }

        isLoading = false;
    }

    async function save() {
        if (isSaving || !hasChanges) {
            return;
        }

        isSaving = true;

        try {
            const settings = await ApiClient.settings.update(CommonHelper.filterRedactedProps(formSettings));
            init(settings);
            setErrors({});
            addSuccessToast("Successfully saved files storage settings.");
        } catch (err) {
            ApiClient.errorResponseHandler(err);
        }

        isSaving = false;
    }

    function init(settings = {}) {
        formSettings = {
            s3: settings?.s3 || {},
        };
        originalFormSettings = JSON.parse(JSON.stringify(formSettings));
    }

    function reset() {
        formSettings = JSON.parse(JSON.stringify(originalFormSettings || {}));
    }
</script>

<SettingsSidebar />

<PageWrapper>
    <header class="page-header">
        <nav class="breadcrumbs">
            <div class="breadcrumb-item">Settings</div>
            <div class="breadcrumb-item">{$pageTitle}</div>
        </nav>
    </header>

    <div class="wrapper">
        <form class="panel" autocomplete="off" on:submit|preventDefault={() => save()}>
            <div class="content txt-xl m-b-base">
                <p>By default PocketBase uses the local file system to store uploaded files.</p>
                <p>
                    If you have limited disk space, you could optionally connect to a S3 compatible storage.
                </p>
            </div>

            {#if isLoading}
                <div class="loader" />
            {:else}
                <Field class="form-field form-field-toggle" let:uniqueId>
                    <input type="checkbox" id={uniqueId} required bind:checked={formSettings.s3.enabled} />
                    <label for={uniqueId}>Use S3 storage</label>
                </Field>

                {#if originalFormSettings.s3?.enabled != formSettings.s3.enabled}
                    <div transition:slide|local={{ duration: 150 }}>
                        <div class="alert alert-warning m-0">
                            <div class="icon">
                                <i class="ri-error-warning-line" />
                            </div>
                            <div class="content">
                                If you have existing uploaded files, you'll have to migrate them manually from
                                the
                                <strong>
                                    {originalFormSettings.s3?.enabled ? "S3 storage" : "local file system"}
                                </strong>
                                to the
                                <strong>{formSettings.s3.enabled ? "S3 storage" : "local file system"}</strong
                                >.
                                <br />
                                There are numerous command line tools that can help you, such as:
                                <a
                                    href="https://github.com/rclone/rclone"
                                    target="_blank"
                                    rel="noopener"
                                    class="txt-bold"
                                >
                                    rclone
                                </a>,
                                <a
                                    href="https://github.com/peak/s5cmd"
                                    target="_blank"
                                    rel="noopener"
                                    class="txt-bold"
                                >
                                    s5cmd
                                </a>, etc.
                            </div>
                        </div>
                        <div class="clearfix m-t-base" />
                    </div>
                {/if}

                {#if formSettings.s3.enabled}
                    <div class="grid" transition:slide|local={{ duration: 150 }}>
                        <div class="col-lg-12">
                            <Field class="form-field required" name="s3.endpoint" let:uniqueId>
                                <label for={uniqueId}>Endpoint</label>
                                <input
                                    type="text"
                                    id={uniqueId}
                                    required
                                    bind:value={formSettings.s3.endpoint}
                                />
                            </Field>
                        </div>
                        <div class="col-lg-6">
                            <Field class="form-field required" name="s3.bucket" let:uniqueId>
                                <label for={uniqueId}>Bucket</label>
                                <input
                                    type="text"
                                    id={uniqueId}
                                    required
                                    bind:value={formSettings.s3.bucket}
                                />
                            </Field>
                        </div>
                        <div class="col-lg-6">
                            <Field class="form-field required" name="s3.region" let:uniqueId>
                                <label for={uniqueId}>Region</label>
                                <input
                                    type="text"
                                    id={uniqueId}
                                    required
                                    bind:value={formSettings.s3.region}
                                />
                            </Field>
                        </div>
                        <div class="col-lg-6">
                            <Field class="form-field required" name="s3.accessKey" let:uniqueId>
                                <label for={uniqueId}>Access key</label>
                                <input
                                    type="text"
                                    id={uniqueId}
                                    required
                                    bind:value={formSettings.s3.accessKey}
                                />
                            </Field>
                        </div>
                        <div class="col-lg-6">
                            <Field class="form-field required" name="s3.secret" let:uniqueId>
                                <label for={uniqueId}>Secret</label>
                                <RedactedPasswordInput
                                    id={uniqueId}
                                    required
                                    bind:value={formSettings.s3.secret}
                                />
                            </Field>
                        </div>
                        <div class="col-lg-12">
                            <Field class="form-field" name="s3.forcePathStyle" let:uniqueId>
                                <input
                                    type="checkbox"
                                    id={uniqueId}
                                    bind:checked={formSettings.s3.forcePathStyle}
                                />
                                <label for={uniqueId}>
                                    <span class="txt">Force path-style addressing</span>
                                    <i
                                        class="ri-information-line link-hint"
                                        use:tooltip={{
                                            text: 'Forces the request to use path-style addressing, eg. "https://s3.amazonaws.com/BUCKET/KEY" instead of the default "https://BUCKET.s3.amazonaws.com/KEY".',
                                            position: "top",
                                        }}
                                    />
                                </label>
                            </Field>
                        </div>
                        <!-- margin helper -->
                        <div class="col-lg-12" />
                    </div>
                {/if}

                <div class="flex">
                    <div class="flex-fill" />
                    {#if hasChanges}
                        <button
                            type="button"
                            class="btn btn-secondary btn-hint"
                            disabled={isSaving}
                            on:click={() => reset()}
                        >
                            <span class="txt">Cancel</span>
                        </button>
                    {/if}
                    <button
                        type="submit"
                        class="btn btn-expanded"
                        class:btn-loading={isSaving}
                        disabled={!hasChanges || isSaving}
                        on:click={() => save()}
                    >
                        <span class="txt">Save changes</span>
                    </button>
                </div>
            {/if}
        </form>
    </div>
</PageWrapper>

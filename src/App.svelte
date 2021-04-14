<script lang="ts">
  import {
    Content,
    Breadcrumb,
    BreadcrumbItem,
    Button,
    Grid,
    Row,
    Column,
    StructuredListSkeleton,
    StructuredList,
    StructuredListBody,
    StructuredListHead,
    StructuredListCell,
    StructuredListRow,
  } from 'carbon-components-svelte';
  import Theme from './components/Theme.svelte';
  import {
    ChildHandshake,
    WindowMessenger,
    DebugMessenger,
    RemoteHandle,
    LocalHandle,
    debug,
  } from 'post-me';
  import type { Permission } from 'skynet-mysky-utils';

  // Permissions Types and Methods for post-me to call

  type PermissionStatus = {
    accepted: Permission[];
    rejected: Permission[];
  };

  type ParentMethods = {
    // parent must expose this method.
    setPermissions: (permissions: PermissionStatus) => void;
  };

  type ParentEvents = {};

  type ChildMethods = {
    // child exposes this method to the parent, requesting currently permitted & denied permissions
    promptPermissions: (
      permitted: Permission[],
      denied: Permission[]
    ) => string;
  };

  type ChildEvents = {};

  let theme: 'g10' = 'g10';

  // localState, promptPermitted and promptDenied are set using argustments to "promptPermissions"
  let promptPermitted: Permission[] | null = null;
  let promptDenied: Permission[] | null = null;

  // array of i of checkboxes selected
  let selectedItems: number[] = [];

  // initial load, true until promptPermissions invoked by parent
  let loading: boolean = true;

  // true once submitted when parent's setPermissions is invoked by this child.
  let submitted: boolean = false;

  // more post-me things
  let remoteHandle: RemoteHandle<ParentMethods, ParentEvents> | null = null;
  let localHandle: LocalHandle<ChildMethods, ChildEvents> | null = null;

  // method exposed to parent to call after handshake.
  const model = {
    promptPermissions: (permitted: Permission[], denied: Permission[]) => {
      loading = false;
      console.log('prompted permitted', permitted);
      console.log('prompted denied', denied);

      promptPermitted = [...permitted];
      promptDenied = [...denied];

      return 'prompting user...';
    },
  };

  // init iframe for async stuff
  async function init() {
    // Establish handshake with parent skapp.

    let messenger = new WindowMessenger({
      localWindow: window,
      remoteWindow: window.parent,
      remoteOrigin: '*',
    });

    // Optional debug all the low level messages echanged
    const log = debug('post-me:child');
    messenger = DebugMessenger(messenger, log);

    const connection = await ChildHandshake(messenger, model);

    remoteHandle = connection.remoteHandle();
    localHandle = connection.localHandle();
  }

  // invoke async init.
  init();

  // called when button is pressed.
  function handleClick() {
    // clear out submit button
    submitted = true;

    // get list of now "checked" items from items that were previously "denied"
    let accepted = promptDenied.filter((permission, i) => {
      return selectedItems.includes(i);
    });

    // get list of items not checked that were previously "denied"
    let rejected = promptDenied.filter((permission, i) => {
      return !selectedItems.includes(i);
    });

    // add already permitted to newly accepted
    accepted = [...promptPermitted, ...accepted];

    //call parent's exposed "setPermissions" method
    remoteHandle.call('setPermissions', {
      accepted,
      rejected,
    });
  }
</script>

<Theme persist bind:theme>
  <Content style="background: none; padding: 1rem">
    <Grid>
      <Row>
        <Column lg={16}>
          <Breadcrumb noTrailingSlash aria-label="Page navigation">
            <BreadcrumbItem>MySky Permissions</BreadcrumbItem>
          </Breadcrumb>
          <h1 style="margin-bottom: 1.5rem">Permissions Request</h1>
        </Column>
      </Row>

      {#if loading}
        <Row>
          <Column noGutter>
            <StructuredListSkeleton rows={3} />
          </Column>
        </Row>
        <Row>
          <Button skeleton />
        </Row>
      {/if}

      {#if !loading}
        <Row>
          <Column noGutter>
            <StructuredList>
              <StructuredListHead>
                <StructuredListRow head>
                  <StructuredListCell head>Requestor</StructuredListCell>
                  <StructuredListCell head>Data Domain</StructuredListCell>
                  <StructuredListCell head>File Type</StructuredListCell>
                  <StructuredListCell head>Permissions Type</StructuredListCell>
                  <StructuredListCell head>Allow?</StructuredListCell>
                </StructuredListRow>
              </StructuredListHead>
              <StructuredListBody>
                {#each promptPermitted as prompted, i}
                  <StructuredListRow>
                    <StructuredListCell noWrap
                      >{prompted.requestor}</StructuredListCell
                    >
                    <StructuredListCell>{prompted.path}</StructuredListCell>
                    <StructuredListCell
                      >{prompted.category === 1
                        ? 'Discoverable'
                        : 'Hidden'}</StructuredListCell
                    >
                    <StructuredListCell
                      >{prompted.permType === 4
                        ? 'Read'
                        : 'Write'}</StructuredListCell
                    >
                    <StructuredListCell>
                      <input type="checkbox" checked disabled />
                    </StructuredListCell>
                  </StructuredListRow>
                {/each}
                {#each promptDenied as prompted, i}
                  <StructuredListRow>
                    <StructuredListCell noWrap
                      >{prompted.requestor}</StructuredListCell
                    >
                    <StructuredListCell>{prompted.path}</StructuredListCell>
                    <StructuredListCell
                      >{prompted.category === 1
                        ? 'Discoverable'
                        : 'Hidden'}</StructuredListCell
                    >
                    <StructuredListCell
                      >{prompted.permType === 4
                        ? 'Read'
                        : 'Write'}</StructuredListCell
                    >
                    <StructuredListCell>
                      <input
                        type="checkbox"
                        bind:group={selectedItems}
                        value={i}
                      />
                    </StructuredListCell>
                  </StructuredListRow>
                {/each}
              </StructuredListBody>
            </StructuredList>
          </Column>
        </Row>
        {#if !submitted}
          <Row>
            <Button on:click={handleClick}>Allow Permissions</Button>
          </Row>
        {/if}
        {#if submitted}
          <Row>
            <Button disabled>Allow Permissions</Button>
          </Row>
        {/if}
      {/if}
    </Grid>
  </Content>
</Theme>

<script lang="ts">
  import { Styles, Alert, Tooltip, Accordion, AccordionItem, Col, Row, Button, ButtonGroup, Card, CardBody, CardHeader, CardFooter, Container, Progress, ListGroupItem, ListGroup } from 'sveltestrap';
    import { onMount } from 'svelte';
    //import QrCode from 'svelte-qr-code';
    import { pb } from './lib/pocketbase';
  let roles = [], candidates = [], settings;
  let role;
  let id = Math.random();
  let responses: { [id: string] : string } = {};
  let votesWarning = false;
  
  let disableSubmit = false;
  let closeAll = null;
  let confirmVote = false;

  let title = "";
  let society = "";
  let ready = "";

  onMount(async () => {
    const rolesResponse = await pb.collection('roles').getList(1,50, {
      sort: 'name',
      filter: 'active=true',
    });
    roles = rolesResponse.items;
    const candidatesResponse = await pb.collection('candidates').getList(1,50, {
      sort: 'type,name',
      expand: 'roles',
    });
    candidates = candidatesResponse.items;
    const settingsResponse = await pb.collection('settings').getList(1,50, {
      sort: 'field',
    });
    settings = settingsResponse.items;

    pb.collection('settings').subscribe('*', async ({action, record}) => {
      if (action === 'update') {
        if (record.field === 'ready') {
          ready = record.value;
        } else if (record.field === 'title') {
          title = record.value;
        } else if (record.field === 'society') {
          society = record.value;
        }
      }
  });

    title = settings.find(setting => setting.field === 'title').value;
    ready = settings.find(setting => setting.field === 'ready').value;
    society = settings.find(setting => setting.field === 'society').value;
  });

  async function vote(roleInput, candidateInput) {
    const data = {
      role: roleInput,
      candidate: candidateInput,
      rand: id,
    };
    const response = await pb.collection('votes').create(data, {$autoCancel: false});
    console.log(response);
  }

  async function castVotes() {
    if (Object.keys(responses).length === roles.length) {
      for (role in roles) {
        console.log(roles[role], responses[roles[role].name]);
        try {
          vote(roles[role].id, responses[roles[role].name]);
        } catch (e) {
          console.log(e);
        }
      }
    } else {
      votesWarning = true;
      
    }
    closeAll = false;
    disableSubmit = true;
    confirmVote = true;
    ready = "false";

  }
  for (role in roles) {
    responses[role] = null;
  }
</script>

<main>
  <Styles />
  <Card>
    <CardHeader>
      <h1>{title}</h1>
    </CardHeader>

    <Alert color="success" style="margin-top: 1em;" isOpen={confirmVote}>
          Your votes have been cast. Please close this site.
    </Alert>

    {#if ready === "true"}
    <CardBody>
        <Alert color="warning" isOpen={votesWarning} toggle={() => (votesWarning = false)}>
          Please ensure you have selected a vote for all {roles.length} candidates.
        </Alert>

        

        <Row class="d-grid gap-2 col-12 mx-auto" style="padding-bottom: 1em;"><Button color={closeAll ? "danger" : "success"} on:click={() => closeAll=!closeAll}>{!closeAll ? "Show" : "Hide"} All Candidates</Button></Row>

        <Accordion stayOpen>
          {#each roles as role}
            {#if role.active == true}
              <AccordionItem active={closeAll} header="{role.name.charAt(0).toUpperCase() + role.name.slice(1)}">
                <ListGroup class="mx-auto">
                  {#each candidates as candidate (candidate.id)}
                    {#if candidate.running_for.includes(role.id)}
                      <ListGroupItem class="col-12 mx-auto" tag="button" color="{candidate.type == "human" ? "light" : "warning"}" active={responses[role.name] === candidate.id} on:click={() => {responses[role.name] = candidate.id;}}>
                          {candidate.name}
                      </ListGroupItem>
                    {/if}
                  {/each}
                    </ListGroup>
              </AccordionItem>
            {/if}
          {/each}
        </Accordion>
      
    </CardBody>

      <div class="d-grid gap-2 col-12 mx-auto" style="padding-bottom: 1em;">
        <Row>
          <div class="tooltip-wrapper disabled" id="submit-btn-wrapper">
            <Button color={Object.keys(responses).length == roles.length ? "success" : "danger"} class="btn btn-default" id="submit-btn" disabled={disableSubmit || Object.keys(responses).length != roles.length} on:click={() => castVotes()}>Submit</Button>
          </div>
          <Tooltip target="submit-btn-wrapper" placement="bottom">
            {#if Object.keys(responses).length != roles.length}
              Please select a vote for all {roles.length} candidates.
            {:else}
              {#if disableSubmit}
                You have already submitted your votes.
              {:else}
                Submit your votes.
              {/if}
            {/if}
          </Tooltip>
        </Row>
          <div class="text-center">{Object.keys(responses).length} of {roles.length}</div>
          <Progress color="success" value={Object.keys(responses).length} max={roles.length} />
      </div>
    {/if}

    <Alert isOpen={(ready=="false" && !confirmVote)} color="warning" style="margin-top:1em;margin-bottom: 1em;">
      Voting is not yet open. Please check back later.
    </Alert>
        <CardFooter>
        <Row class="d-grid gap-2 col-12 mx-auto">
        <!-- <Col>
        </Col>
        <Col class="mx-auto"> -->
          <Container>Designed by Kieran Roper</Container>
        <!-- </Col>
        <Col>
        </Col> -->
      </Row>
      <Row class="d-grid gap-2 col-12 mx-auto">
        {society}
      </Row>
      <Button size="sm" style="margin-top:0.5em;" >Admin</Button>
  </CardFooter>
  </Card>
</main>

<style>
  .tooltip-wrapper .btn[disabled] {
    display: inline-block;
  /* don't let button block mouse events from reaching wrapper */
  pointer-events: none;
}

.tooltip-wrapper.disabled {
  /* OPTIONAL pointer-events setting above blocks cursor setting, so set it here */
  display: inline-block;
  cursor: not-allowed;
}
</style>
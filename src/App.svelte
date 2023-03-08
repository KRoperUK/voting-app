<script lang="ts">
  import { Styles, Alert, InputGroup, Tooltip, Accordion, AccordionItem, Col, Row, Button, ButtonGroup, Card, CardBody, Label, CardHeader, CardFooter, Container, Progress, ListGroupItem, ListGroup, Modal, ModalBody, Input, ModalHeader, ModalFooter } from 'sveltestrap';
    import { onMount } from 'svelte';
    //import QrCode from 'svelte-qr-code';
    import { pb } from './lib/pocketbase';
    import { each } from 'svelte/internal';
    import Admin from './lib/Admin.svelte';
    import {GeoJSON, LeafletMap, TileLayer} from 'svelte-leafletjs';
  let roles = [], candidates = [], settings;
  let role;
  let id = Math.random();
  let responses: { [id: string] : string } = {};
  let votesWarning = false;

  let idInput;
  let idReturn = [];

  let memberID;

  let unauthenticated = true;
  let preventionAnswer;

  let adminModalOpen = false;

  let memberCSVModal = false;
  let CSVInput = "";
  
  let disableSubmit = false;
  let closeAll = null;
  let confirmVote = false;

  let idReturned = false;

  let title = "";
  let society = "";
  let ready = "";
  let password = "";
  let passwordAttempt;

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
    password = settings.find(setting => setting.field === 'password').value;
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
  async function authenticate(user_id) {
    memberID = user_id;
    unauthenticated = false;
  }

  async function addMember(surname, forename, id_number) {
    const data = {
      surname: surname,
      forename: forename,
      id_number: id_number,
    };
    const response = await pb.collection('members').create(data, {$autoCancel: false});
    console.log(response);
  }

  async function attemptAuthentication() {
    const idResponse = await pb.collection('members').getList(1,50, {
      sort: "surname",
      filter: "id_number='" + idInput + "'",
    });
    idReturn = idResponse.items;
    idReturned = true;
  }

  async function handleMemberCSV() {
    let CSVArray = CSVInput.split("\n");
    CSVArray.forEach(csvline => {
      let memberInfo = csvline.split("^");
      memberInfo[0] = memberInfo[0].replaceAll("\"","");
      let student_id = memberInfo[1].replaceAll("\"","");
      let names = memberInfo[0].split(',');
      addMember(names[0], names[1], student_id);
    });
  }
  
  async function preventMember() {
    const preventionResponse = await pb.collection('members').getList(1,2, {
      filter: "id='" + memberID + "\'",
    });
    preventionAnswer = preventionResponse.items;
    if (preventionAnswer[0].voted) {
      return false;
    } else {
      const data = {
        voted: true,
      };
      const response = await pb.collection('members').update(memberID, data);
      console.log(response);
      return true;
    }
  }

  async function castVotes() {
    if (preventMember()){
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

  }
  for (role in roles) {
    responses[role] = null;
  }
</script>

<svelte:head>
  <title>{title}</title>
</svelte:head>

<main>
  <Styles />
  <Card>
    <CardHeader>
      <h1>{title}</h1>
    </CardHeader>
    
    
    <Modal id="adminModal" isOpen={adminModalOpen}>
      <ModalHeader>
        <h1>Admin Dashboard</h1>
      </ModalHeader>
      <ModalBody>
        <Row>
          <div class="d-grid gap-2 col-12 mx-auto">
            <h3>General Settings</h3>
            <Row>
              <Button color="primary" on:click={() => closeAll = true}>Close Voting</Button>
            </Row>
            <Row>
              <Button color="primary" on:click={() => closeAll = false}>Open Voting</Button>
            </Row>
            <p>Lorem ipsum</p>
          </div>
        </Row>
        <Row>
          <h3>Candidate Settings</h3>
          <p>Lorem ipsum</p>
        </Row>
        <Row>
          <h3>Other Settings</h3>
          <p>Lorem ipsum</p>
        </Row>
      </ModalBody>
      <ModalFooter>
        <Row>
          <Col>
            <Button color="primary" on:click={() => memberCSVModal = true}>Import Members</Button>
          </Col>
          <Col>
            <Button color="success">Save Changes</Button>
          </Col>
          <Col>
            <Button color="danger">Discard Changes</Button>
          </Col>
        </Row>
      </ModalFooter>
    </Modal>
    
    <Modal id="memberCSVImport" isOpen={memberCSVModal}>
      <ModalBody class="d-grid gap-2 col-12 mx-auto">
        <h1>Member CSV Import</h1>
        <Input type="textarea" name="memberCSVImportText" bind:value={CSVInput} id="memberCSVImportText" />
        <Button on:click={() => handleMemberCSV()}>Import</Button>
      </ModalBody>
    </Modal>
    
    <Alert color="success" style="margin-top: 1em;" isOpen={confirmVote}>
      Your votes have been cast. Please close this site.
    </Alert>
    {#if unauthenticated}
    <Card style="margin-top: 0.5em;margin-bottom:0.5em;" class="d-grid gap-2 col-12 mx-auto">
      <Alert color="warning" style="margin-top:0.5em;">
        You must authenticate.
      </Alert>
      <Input placeholder="Student ID Number" bind:value={idInput} />
      <Input type="password" placeholder="AGM Password" bind:value={passwordAttempt} />
      
      {#if password == passwordAttempt}
      <Button on:click={() => attemptAuthentication()}>Search</Button>
      {:else}
      <Label>Enter the given password.</Label>
      {/if}
      
      {#if idReturned}
      <Row>
        <Col>
          <Card color="success" class="text-light" style="padding: 0.1em; margin-bottom: 0.3rem;">Results:</Card>
        </Col>
      </Row>
      {#if idReturn.length === 0}
      <Row>
        <Card color="danger">
          <Label class="text-light" >No results found.</Label>
        </Card>
      </Row>
      {/if}
        <Row class="mx-auto">
        {#each idReturn as eachID (eachID.surname)}
        <Col>
          <Card style="padding: 1em;" color="{eachID.voted ? "danger" : "light"}">
            <Container><Label class="{eachID.voted ? "text-light" : "text-dark"}" >{eachID.forename} {eachID.surname}</Label></Container>
                {#if !eachID.voted}
                <Button color="success" on:click={() => authenticate(eachID.id)}>Login</Button>
                {:else}
                <Button >Already voted.</Button>
                {/if}
              </Card>
            </Col> 
        {/each}
        </Row>
      {/if}
    </Card>

    {:else}

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
      <Button size="sm" style="margin-top:0.5rem;margin-bottom:0.5rem;" on:click={() => {adminModalOpen=true}} >Admin</Button>
  </CardFooter>
  </Card>
</main>

<style>
.tooltip-wrapper.disabled {
  /* OPTIONAL pointer-events setting above blocks cursor setting, so set it here */
  display: inline-block;
  cursor: not-allowed;
}
</style>
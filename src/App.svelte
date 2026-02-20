<script lang="ts">
  type WorkEntry = { company: string; date: string };
  type EducationEntry = { degree: string; date: string };

  let firstName = '';
  let lastName = '';
  let email = '';
  let phone = '';
  let location = '';

  let workEntries: WorkEntry[] = [
    { company: '', date: '' },
    { company: '', date: '' },
    { company: '', date: '' },
  ];

  function addWork() {
    workEntries = [...workEntries, { company: '', date: '' }];
  }

  function removeWork(index: number) {
    workEntries = workEntries.filter((_, i) => i !== index);
  }

  let educationEntries: EducationEntry[] = [
    { degree: '', date: '' },
  ];

  function addEducation() {
    educationEntries = [...educationEntries, { degree: '', date: '' }];
  }

  function removeEducation(index: number) {
    educationEntries = educationEntries.filter((_, i) => i !== index);
  }

  let jobDescription = '';
</script>

<main>
  <div class="page">
    <h1 class="page-title">Profile Form</h1>

    <!-- Section 1: Personal Information -->
    <section class="card">
      <h2 class="section-title">
        <span class="section-badge">1</span>
        Personal Information
      </h2>
      <div class="form-grid">
        <div class="field">
          <label for="firstName">First Name</label>
          <input id="firstName" type="text" bind:value={firstName} placeholder="John" />
        </div>
        <div class="field">
          <label for="lastName">Last Name</label>
          <input id="lastName" type="text" bind:value={lastName} placeholder="Doe" />
        </div>
        <div class="field">
          <label for="email">Email</label>
          <input id="email" type="email" bind:value={email} placeholder="john@example.com" />
        </div>
        <div class="field">
          <label for="phone">Phone</label>
          <input id="phone" type="tel" bind:value={phone} placeholder="+1 (555) 000-0000" />
        </div>
        <div class="field field--full">
          <label for="location">Location</label>
          <input id="location" type="text" bind:value={location} placeholder="City, State / Country" />
        </div>
      </div>
    </section>

    <!-- Section 2: Work Experience -->
    <section class="card">
      <div class="section-header">
        <h2 class="section-title">
          <span class="section-badge">2</span>
          Work Experience
        </h2>
        <button class="btn-add" type="button" on:click={addWork}>
          <span class="btn-add-icon">+</span> Add Position
        </button>
      </div>
      {#each workEntries as entry, i}
        <div class="row-grid row-divider">
          <div class="row-label">Position {i + 1}</div>
          <div class="field">
            <label for="company-{i}">Company Name</label>
            <input id="company-{i}" type="text" bind:value={entry.company} placeholder="Acme Corp" />
          </div>
          <div class="field">
            <label for="work-date-{i}">Date</label>
            <input id="work-date-{i}" type="text" bind:value={entry.date} placeholder="e.g. Jan 2020 – Mar 2023" />
          </div>
          <button
            class="btn-remove"
            type="button"
            title="Remove position"
            disabled={workEntries.length === 1}
            on:click={() => removeWork(i)}
          >×</button>
        </div>
      {/each}
    </section>

    <!-- Section 3: Education -->
    <section class="card">
      <div class="section-header">
        <h2 class="section-title">
          <span class="section-badge">3</span>
          Education
        </h2>
        <button class="btn-add" type="button" on:click={addEducation}>
          <span class="btn-add-icon">+</span> Add Entry
        </button>
      </div>
      {#each educationEntries as entry, i}
        <div class="row-grid row-divider">
          <div class="row-label">Entry {i + 1}</div>
          <div class="field">
            <label for="degree-{i}">Degree</label>
            <input id="degree-{i}" type="text" bind:value={entry.degree} placeholder="Bachelor of Science" />
          </div>
          <div class="field">
            <label for="edu-date-{i}">Date</label>
            <input id="edu-date-{i}" type="text" bind:value={entry.date} placeholder="e.g. May 2018" />
          </div>
          <button
            class="btn-remove"
            type="button"
            title="Remove entry"
            disabled={educationEntries.length === 1}
            on:click={() => removeEducation(i)}
          >×</button>
        </div>
      {/each}
    </section>

    <!-- Section 4: Job Description -->
    <section class="card">
      <h2 class="section-title">
        <span class="section-badge">4</span>
        Job Description
      </h2>
      <div class="field">
        <label for="jobDescription">Describe the role or position you are applying for</label>
        <textarea
          id="jobDescription"
          bind:value={jobDescription}
          placeholder="Paste or write the job description here…"
          rows="8"
        ></textarea>
      </div>
    </section>
  </div>
</main>

<style>
  main {
    width: 100%;
    min-height: 100vh;
    background: #f0f2f5;
    display: flex;
    justify-content: center;
    padding: 2.5rem 1rem 4rem;
    box-sizing: border-box;
  }

  .page {
    width: 100%;
    max-width: 760px;
  }

  .page-title {
    font-size: 1.75rem;
    font-weight: 700;
    color: #111827;
    margin: 0 0 1.75rem;
    letter-spacing: -0.02em;
  }

  /* Card */
  .card {
    background: #ffffff;
    border-radius: 14px;
    border: 1px solid #e5e7eb;
    padding: 1.75rem 2rem;
    margin-bottom: 1.25rem;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.06);
  }

  /* Section title */
  .section-title {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    font-size: 1.05rem;
    font-weight: 600;
    color: #111827;
    margin: 0 0 1.5rem;
  }

  .section-badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 26px;
    height: 26px;
    border-radius: 50%;
    background: #4f46e5;
    color: #fff;
    font-size: 0.78rem;
    font-weight: 700;
    flex-shrink: 0;
  }

  /* Grids */
  .form-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem 1.25rem;
  }

  .field--full {
    grid-column: 1 / -1;
  }

  .section-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 1.5rem;
  }

  .section-header .section-title {
    margin: 0;
  }

  .row-grid {
    display: grid;
    grid-template-columns: 90px 1fr 1fr 32px;
    gap: 0.75rem 1.25rem;
    align-items: end;
    padding: 0.75rem 0;
  }

  .row-divider + .row-divider {
    border-top: 1px solid #f3f4f6;
  }

  .row-label {
    font-size: 0.78rem;
    font-weight: 600;
    color: #6b7280;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    padding-bottom: 0.6rem;
  }

  /* Fields */
  .field {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
  }

  label {
    font-size: 0.82rem;
    font-weight: 500;
    color: #374151;
  }

  input {
    height: 40px;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    padding: 0 0.75rem;
    font-size: 0.9rem;
    color: #111827;
    background: #f9fafb;
    transition: border-color 0.18s, box-shadow 0.18s, background 0.18s;
    outline: none;
    width: 100%;
    box-sizing: border-box;
  }

  input::placeholder {
    color: #9ca3af;
  }

  input:focus {
    border-color: #4f46e5;
    background: #fff;
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.12);
  }

  textarea {
    border: 1px solid #d1d5db;
    border-radius: 8px;
    padding: 0.65rem 0.75rem;
    font-size: 0.9rem;
    font-family: inherit;
    color: #111827;
    background: #f9fafb;
    resize: vertical;
    transition: border-color 0.18s, box-shadow 0.18s, background 0.18s;
    outline: none;
    width: 100%;
    box-sizing: border-box;
    line-height: 1.55;
  }

  textarea::placeholder {
    color: #9ca3af;
  }

  textarea:focus {
    border-color: #4f46e5;
    background: #fff;
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.12);
  }

  /* Actions */
  .actions {
    display: flex;
    justify-content: flex-end;
    gap: 0.75rem;
    margin-top: 0.5rem;
  }

  button {
    height: 40px;
    padding: 0 1.4rem;
    border-radius: 8px;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.18s, border-color 0.18s;
    border: 1px solid transparent;
  }

  .btn-primary {
    background: #4f46e5;
    color: #fff;
    border-color: #4f46e5;
  }

  .btn-primary:hover {
    background: #4338ca;
    border-color: #4338ca;
  }

  .btn-secondary {
    background: #fff;
    color: #374151;
    border-color: #d1d5db;
  }

  .btn-secondary:hover {
    background: #f3f4f6;
  }

  .btn-add {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
    height: 34px;
    padding: 0 1rem;
    border-radius: 8px;
    font-size: 0.82rem;
    font-weight: 600;
    color: #4f46e5;
    background: #eef2ff;
    border: 1px solid #c7d2fe;
    cursor: pointer;
    transition: background 0.18s, border-color 0.18s;
    white-space: nowrap;
  }

  .btn-add:hover {
    background: #e0e7ff;
    border-color: #a5b4fc;
  }

  .btn-add-icon {
    font-size: 1.1rem;
    line-height: 1;
  }

  .btn-remove {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    border-radius: 6px;
    font-size: 1.1rem;
    line-height: 1;
    color: #9ca3af;
    background: transparent;
    border: 1px solid #e5e7eb;
    cursor: pointer;
    transition: color 0.18s, background 0.18s, border-color 0.18s;
    padding: 0;
    margin-bottom: 4px;
  }

  .btn-remove:hover:not(:disabled) {
    color: #ef4444;
    background: #fef2f2;
    border-color: #fca5a5;
  }

  .btn-remove:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  @media (max-width: 560px) {
    .form-grid {
      grid-template-columns: 1fr;
    }

    .field--full {
      grid-column: 1;
    }

    .row-grid {
      grid-template-columns: 1fr 1fr 32px;
    }

    .row-label {
      grid-column: 1 / -1;
      padding-bottom: 0;
    }
  }
</style>

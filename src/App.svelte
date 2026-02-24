<script lang="ts">
  import { jsPDF } from 'jspdf';
  type WorkEntry = { company: string; date: string; location: string; level: string; bulletPoints: number };
  type EducationEntry = { degree: string; date: string; institution: string; location: string };
  type TechnicalSkillsData = { technicalSkills: { category: string; skills: string[] }[] };
  type GeneratedWorkExperience = { company: string; date: string; sentences: string[] };
  type ResumeData = {
    summary: string;
    formalRole: string;
    jobTitles: string[];
    educationDegrees: string[];
    technicalSkills: TechnicalSkillsData;
    workExperience: GeneratedWorkExperience[];
  };

  const STORAGE_KEY = 'profile-form-data';

  function loadSaved() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) return JSON.parse(raw);
    } catch {}
    return null;
  }

  const saved = loadSaved();

  let fullName = $state<string>(saved?.fullName ?? '');
  let email = $state<string>(saved?.email ?? '');
  let phone = $state<string>(saved?.phone ?? '');
  let location = $state<string>(saved?.location ?? '');

  let workEntries = $state<WorkEntry[]>(saved?.workEntries ?? [
    { company: '', date: '', location: '', level: '', bulletPoints: 5 },
    { company: '', date: '', location: '', level: '', bulletPoints: 4 },
    { company: '', date: '', location: '', level: '', bulletPoints: 3 },
  ]);

  function addWork() {
    workEntries = [...workEntries, { company: '', date: '', location: '', level: '', bulletPoints: 5 }];
  }

  function removeWork(index: number) {
    workEntries = workEntries.filter((_, i) => i !== index);
  }

  function getWorkExperienceSummary(): string {
    return workEntries
      .filter(e => e.company || e.date)
      .map(e => `${e.company}${e.level ? ` (${e.level} level)` : ''}: ${e.date}`)
      .join('\n');
  }

  let educationEntries = $state<EducationEntry[]>(saved?.educationEntries ?? [
    { degree: '', date: '', institution: '', location: '' },
  ]);

  function addEducation() {
    educationEntries = [...educationEntries, { degree: '', date: '', institution: '', location: '' }];
  }

  function removeEducation(index: number) {
    educationEntries = educationEntries.filter((_, i) => i !== index);
  }

  let jobDescription = $state<string>('');

  const OPENAI_API_KEY = import.meta.env.VITE_OPENAI_API_KEY as string;
  const OPENAI_MODEL = import.meta.env.VITE_OPENAI_MODEL as string;

  let isChecking = $state(false);
  let isGeneratingTechnicalSkills = $state(false);
  let isGeneratingExperience = $state(false);
  let eligibilityResult = $state<boolean | null>(null);
  let resumeData = $state<ResumeData | null>(null);

  async function checkEligibilityAndGenerateSummary() {
    eligibilityResult = null;
    isChecking = true;
    try {
      const career = getWorkExperienceSummary();
      const systemPrompt = `You are a job eligibility validator and expert resume writer.
Your tasks are to:
1) Determine whether a resume should be generated for a given Job Description (JD).
2) If eligible, generate a professional summary, a formal role, and job titles for each position based on the JD and the candidate's career.
3) If eligible, update the education degrees based on the candidate's degrees and the JD.

ELIGIBILITY RULES (apply in order):
1. If the JD requires any type of clearance, set "eligible" to false.
2. If the JD requires on-site or hybrid work or specifies a required candidate location,
  - Set "eligible" to true ONLY if there is possibility to work fully remote.
  - Otherwise, set "eligible" to false.
3. If none of the above conditions apply, set "eligible" to true.

SUMMARY RULES (only when eligible is true):
- Length: 3–4 sentences only
- Tone: professional, confident, concise
- Optimization: ATS-friendly (use relevant keywords from the job description)
- Alignment: tailor directly to the provided job description and align with the candidate's experience years.
- Perspective: third person, no personal pronouns
- Formatting: plain text, no bullet points, no headings

FORMAL ROLE RULES (only when eligible is true):
- The title should be 2-4 words long.
- The title should be appropriate for the job description.
- The title should be simple and concise, but common in the industry.

JOB TITLE RULES (only when eligible is true):
- Each job title should be 2-4 words long.
- Each job title should be appropriate for the job description.
- Each job title should be simple and concise, but common in the industry.
- The job titles should be familiar to the career flow.

EDUCATION DEGREE RULES (only when eligible is true):
Only modify the degree if it is not related to the job description.
- Each degree should be appropriate for the job description.
- Each degree should be common in the industry.

Output Rules: Respond ONLY in valid JSON.
JSON schema:
{
  "eligible": Boolean,
  "summary": "Summary text or null if not eligible",
  "formalRole": "Formal role text or null if not eligible",
  "jobTitles": ["Job Title1", "Job Title2", ...] or null if not eligible,
  "educationDegrees": ["Updated Degree1", "Updated Degree2", ...] or null if not eligible
}`;

      const userPrompt = `
Candidate location:
${location}

Here is my brief career.
${career}

Here are my education degrees.
${educationEntries.map(entry => `${entry.degree}`).join('\n')}

Job Description:
${jobDescription}`;

      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`,
        },
        body: JSON.stringify({
          model: OPENAI_MODEL,
          messages: [
            { role: 'system', content: systemPrompt },
            { role: 'user', content: userPrompt },
          ],
          response_format: { type: 'json_object' },
        }),
      });

      const data = await response.json();
      const result = JSON.parse(data.choices[0].message.content);
      eligibilityResult = result.eligible;
      console.log(eligibilityResult);
      return result;
    } finally {
      isChecking = false;
    }
  }

  async function generateTechnicalSkills() {
    isGeneratingTechnicalSkills = true;
    try {
      const systemPrompt = `You are an expert resume writer.
Your task is to generate a professional technical skills section based on the following job description.

Guidelines:
1. List 30-40 technical skills relevant to the job or related fields.
2. Include technologies explicitly mentioned in the job description as well as related skills.
3. Include technologies released during the work period.
4. Categorize skills.
5. Ensure ATS compatibility:
   - Use proper keywords
   - Avoid special symbols or unnecessary formatting
   - Keep technical symbols like '/' or '-' when appropriate (e.g., "CI/CD", "T-SQL")
6. Double-check that all technologies in the job description are included.

Output ONLY the Technical Skills section in JSON format.
JSON schema:
{
  "technicalSkills": [
    {
      "category": "Category1",
      "skills": ["Skill1", "Skill2", "Skill3"]
    }
  ]
}`;

      const userPrompt = `Job Description:
${jobDescription}`;

      const response = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${OPENAI_API_KEY}`,
        },
        body: JSON.stringify({
          model: OPENAI_MODEL,
          messages: [
            { role: 'system', content: systemPrompt },
            { role: 'user', content: userPrompt },
          ],
          response_format: { type: 'json_object' },
        }),
      });

      const data = await response.json();
      const technicalSkills = JSON.parse(data.choices[0].message.content) as TechnicalSkillsData;
      return technicalSkills;
    } finally {
      isGeneratingTechnicalSkills = false;
    }
  }

  async function generateWorkExperience(): Promise<GeneratedWorkExperience[]> {
    isGeneratingExperience = true;
    const results: GeneratedWorkExperience[] = [];
    try {
      for (const entry of workEntries) {
        const systemPrompt = `You are an expert resume writer and ATS optimization specialist.
Your task is to generate professional resume experience sentences based on the provided job description, company information and number of sentences required.

GUIDELINES:
1. Write in third-person only without the name, and he or she.
2. Each sentence must be 150-220 characters and contain detailed, technically rich descriptions of your role, specific contributions, and technologies used.
3. Each experience must reference company industry relevance.
4. Each sentence must end with a period.
5. Each sentence only contain the content with no headers, job titles, or company names.
6. Use ATS-friendly technical keywords.
7. Use only technologies that were publicly available during the work period.
8. No sentence may be vague or generic
9. Avoid special characters except "/" or "-" when required (examples: CI/CD, T-SQL).
10. All technologies mentioned in the job description must be included and used correctly in the sentences.
10. All technologies are included only after their first version (or alpha release date).

OUTPUT FORMAT: Return ONLY the sentences as JSON.
JSON schema:
{
  "sentences": ["Sentence1", "Sentence2", "Sentence3"]
}`;

      const userPrompt = `
Company Name: ${entry.company}
Work Period: ${entry.date}
Required Number of Sentences: ${entry.bulletPoints}

JOB DESCRIPTION:
${jobDescription}`;

        const response = await fetch('https://api.openai.com/v1/chat/completions', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${OPENAI_API_KEY}`,
          },
          body: JSON.stringify({
            model: OPENAI_MODEL,
            messages: [
              { role: 'system', content: systemPrompt },
              { role: 'user', content: userPrompt },
            ],
            response_format: { type: 'json_object' },
          }),
        });

        const data = await response.json();
        const parsed = JSON.parse(data.choices[0].message.content) as { sentences: string[] };
        results.push({ company: entry.company, date: entry.date, sentences: parsed.sentences });
      }
    } finally {
      isGeneratingExperience = false;
    }
    return results;
  }

  async function handleGenerate() {
    resumeData = null;
    const result = await checkEligibilityAndGenerateSummary();
    if (result?.eligible === true) {
      const technicalSkills = await generateTechnicalSkills();
      if (technicalSkills) {
        const workExperience = await generateWorkExperience();
        resumeData = {
          summary: result.summary,
          formalRole: result.formalRole,
          jobTitles: result.jobTitles,
          educationDegrees: result.educationDegrees,
          technicalSkills,
          workExperience,
        };
        downloadResume();
      }
    }
  }

  function downloadResume() {
    const doc = buildResumePdf();
    const safeName = (fullName || 'resume').replace(/\s+/g, '_');
    doc.save(`${safeName}_resume.pdf`);
  }

  function buildResumePdf(): jsPDF {
    const data = resumeData!;
    const doc = new jsPDF({ unit: 'in', format: 'letter', orientation: 'portrait' });

    const margin   = 0.4;
    const pageW    = 8.5;
    const pageH    = 11;
    const contentW = pageW - margin * 2;
    let y = margin;

    const NAME_PT = 22;
    const BODY_PT = 11;
    const LH = (BODY_PT * 1.5) / 72; // line height in inches ≈ 0.2292"

    function checkPage(needed: number) {
      if (y + needed > pageH - margin) {
        doc.addPage();
        y = margin;
      }
    }

    function body(style: 'normal' | 'bold' | 'italic' | 'bolditalic' = 'normal') {
      doc.setFont('helvetica', style);
      doc.setFontSize(BODY_PT);
    }

    function splitBody(text: string, maxWidth: number): string[] {
      doc.setFontSize(BODY_PT);
      return doc.splitTextToSize(text, maxWidth) as string[];
    }

    // ── Header ───────────────────────────────────────────────────────────────
    const nameDisplay = fullName || 'Your Name';
    doc.setFont('helvetica', 'bold');
    doc.setFontSize(NAME_PT);
    const nameLineH = (NAME_PT * 1.5) / 72;

    doc.setTextColor(17, 17, 17);
    doc.text(nameDisplay, margin, y + nameLineH * 0.78);

    if (data.formalRole) {
      const nameW = doc.getTextWidth(nameDisplay);
      body('normal');
      doc.setTextColor(80, 80, 80);
      doc.text('  ' + data.formalRole, margin + nameW, y + nameLineH * 0.78);
    }

    y += nameLineH;

    const contactParts = [email, phone, location].filter(Boolean);
    if (contactParts.length) {
      body('normal');
      doc.setTextColor(80, 80, 80);
      doc.text(contactParts.join('  |  '), margin, y + LH * 0.78);
      y += LH;
    }

    doc.setDrawColor(34, 34, 34);
    doc.setLineWidth(0.02);
    doc.line(margin, y + LH * 0.25, margin + contentW, y + LH * 0.25);
    y += LH;

    doc.setTextColor(17, 17, 17);

    function drawSectionTitle(title: string) {
      checkPage(LH * 2);
      body('bold');
      doc.setTextColor(17, 17, 17);
      doc.text(title.toUpperCase(), margin, y + LH * 0.78);
      y += LH * 1.25;
    }

    // ── Summary ──────────────────────────────────────────────────────────────
    if (data.summary) {
      drawSectionTitle('Professional Summary');
      body('normal');
      splitBody(data.summary, contentW).forEach(line => {
        checkPage(LH);
        doc.text(line, margin, y + LH * 0.78);
        y += LH;
      });
      y += LH;
    }

    // ── Technical Skills ─────────────────────────────────────────────────────
    if (data.technicalSkills.technicalSkills.length) {
      drawSectionTitle('Technical Skills');
      data.technicalSkills.technicalSkills.forEach(cat => {
        const label    = cat.category + ': ';
        const skillsStr = cat.skills.join(', ');

        body('bold');
        const labelW = doc.getTextWidth(label);

        body('normal');
        const firstLineSkills = doc.splitTextToSize(skillsStr, contentW - labelW) as string[];
        const rest = firstLineSkills.length > 1
          ? doc.splitTextToSize(firstLineSkills.slice(1).join(' '), contentW) as string[]
          : [];

        checkPage(LH);
        body('bold');
        doc.text(label, margin, y + LH * 0.78);
        body('normal');
        doc.text(firstLineSkills[0], margin + labelW, y + LH * 0.78);
        y += LH;

        rest.forEach(line => {
          checkPage(LH);
          doc.text(line, margin, y + LH * 0.78);
          y += LH;
        });
      });
      y += LH;
    }

    // ── Work Experience ───────────────────────────────────────────────────────
    if (data.workExperience.length) {
      drawSectionTitle('Work Experience');
      data.workExperience.forEach((job, i) => {
        const entry = workEntries[i];
        checkPage(LH * 2);

        // Row 1: Company (bold, left)  |  Date (normal, right)
        // Row 2: Job Title (normal, left)  |  Location (italic, right)
        const jobTitle = data.jobTitles?.[i] ?? '';

        body('bold');
        doc.text(job.company, margin, y + LH * 0.78);
        body('normal');
        doc.setTextColor(80, 80, 80);
        doc.text(job.date, margin + contentW, y + LH * 0.78, { align: 'right' });
        doc.setTextColor(17, 17, 17);
        y += LH;

        if (jobTitle || entry?.location) {
          if (jobTitle) {
            body('normal');
            doc.text(jobTitle, margin, y + LH * 0.78);
          }
          if (entry?.location) {
            body('italic');
            doc.setTextColor(80, 80, 80);
            doc.text(entry.location, margin + contentW, y + LH * 0.78, { align: 'right' });
            doc.setTextColor(17, 17, 17);
          }
          y += LH;
        }

        // Bullet points
        body('normal');
        const bulletIndent = margin + 0.18;
        const bulletW = contentW - 0.18;
        job.sentences.forEach(sentence => {
          const lines = splitBody(sentence, bulletW);
          checkPage(lines.length * LH);
          lines.forEach((line, li) => {
            if (li === 0) doc.text('\u2022', margin + 0.04, y + LH * 0.78);
            doc.text(line, bulletIndent, y + LH * 0.78);
            y += LH;
          });
        });
        y += LH * 0.25;
      });
      y += LH;
    }

    // ── Education ────────────────────────────────────────────────────────────
    const validEduIndices = educationEntries
      .map((e, i) => ({ e, i }))
      .filter(({ e }) => e.degree || e.institution);
    const validEdu = validEduIndices.map(({ e }) => e);
    if (validEdu.length) {
      drawSectionTitle('Education');
      validEduIndices.forEach(({ e: edu, i: origIdx }) => {
        checkPage(LH * 2);

        const displayDegree = data.educationDegrees?.[origIdx] || edu.degree || '';
        body('bold');
        doc.text(displayDegree, margin, y + LH * 0.78);
        body('normal');
        doc.setTextColor(80, 80, 80);
        doc.text(edu.date || '', margin + contentW, y + LH * 0.78, { align: 'right' });
        doc.setTextColor(17, 17, 17);
        y += LH;

        if (edu.institution || edu.location) {
          body('normal');
          doc.setTextColor(85, 85, 85);
          if (edu.institution) doc.text(edu.institution, margin, y + LH * 0.78);
          if (edu.location) doc.text(edu.location, margin + contentW, y + LH * 0.78, { align: 'right' });
          doc.setTextColor(17, 17, 17);
          y += LH;
        }
        y += LH * 0.25;
      });
    }

    return doc;
  }


  $effect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify({
      fullName,
      email,
      phone,
      location,
      workEntries: $state.snapshot(workEntries),
      educationEntries: $state.snapshot(educationEntries),
    }));
  });
</script>

{#if eligibilityResult !== null}
  <div class="alert-banner {eligibilityResult ? 'alert-eligible' : 'alert-ineligible'}" role="alert">
    <span class="alert-icon">{eligibilityResult ? '✓' : '✕'}</span>
    <span class="alert-text">
      {#if eligibilityResult}
        You appear to meet the requirements for this position. You're good to proceed.
      {:else}
        This position may not be a match based on your profile or the job's requirements.
      {/if}
    </span>
    <button class="alert-close" type="button" aria-label="Dismiss" onclick={() => eligibilityResult = null}>×</button>
  </div>
{/if}

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
          <label for="fullName">Full Name</label>
          <input id="fullName" type="text" bind:value={fullName} placeholder="John Doe" />
        </div>
        <div class="field">
          <label for="email">Email</label>
          <input id="email" type="email" bind:value={email} placeholder="john@example.com" />
        </div>
        <div class="field">
          <label for="phone">Phone</label>
          <input id="phone" type="tel" bind:value={phone} placeholder="+1 (555) 000-0000" />
        </div>
        <div class="field">
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
        <button class="btn-add" type="button" onclick={addWork}>
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
          <div class="field">
            <label for="work-location-{i}">Location</label>
            <input id="work-location-{i}" type="text" bind:value={entry.location} placeholder="City, State / Remote" />
          </div>
          <div class="field">
            <label for="work-level-{i}">Level</label>
            <select id="work-level-{i}" bind:value={entry.level}>
              <option value=""></option>
              <option value="Senior">Senior</option>
              <option value="Mid-Level">Middle</option>
              <option value="Junior">Junior</option>
            </select>
          </div>
          <div class="field field--bullets">
            <label for="work-bullets-{i}">Bullets</label>
            <input id="work-bullets-{i}" type="number" bind:value={entry.bulletPoints} min="2" max="12" />
          </div>
          <button
            class="btn-remove"
            type="button"
            title="Remove position"
            disabled={workEntries.length === 1}
            onclick={() => removeWork(i)}
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
        <button class="btn-add" type="button" onclick={addEducation}>
          <span class="btn-add-icon">+</span> Add Entry
        </button>
      </div>
      {#each educationEntries as entry, i}
        <div class="row-edu row-divider">
          <div class="row-edu-top">
            <div class="row-label">Entry {i + 1}</div>
            <button
              class="btn-remove"
              type="button"
              title="Remove entry"
              disabled={educationEntries.length === 1}
              onclick={() => removeEducation(i)}
            >×</button>
          </div>
          <div class="edu-fields">
            <div class="field">
              <label for="degree-{i}">Degree</label>
              <input id="degree-{i}" type="text" bind:value={entry.degree} placeholder="Bachelor of Science" />
            </div>
            <div class="field">
              <label for="edu-date-{i}">Date</label>
              <input id="edu-date-{i}" type="text" bind:value={entry.date} placeholder="e.g. May 2018" />
            </div>
            <div class="field">
              <label for="edu-institution-{i}">Institution</label>
              <input id="edu-institution-{i}" type="text" bind:value={entry.institution} placeholder="University of Example" />
            </div>
            <div class="field">
              <label for="edu-location-{i}">Location</label>
              <input id="edu-location-{i}" type="text" bind:value={entry.location} placeholder="City, State / Country" />
            </div>
          </div>
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
      <div class="generate-row">
        <button class="btn-generate" type="button" onclick={handleGenerate} disabled={isChecking || isGeneratingTechnicalSkills || isGeneratingExperience}>
          {isChecking ? 'Checking & Generating …' : isGeneratingTechnicalSkills ? 'Building Skills …' : isGeneratingExperience ? 'Writing Experience …' : 'Generate'}
        </button>
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
    line-height: 1.5;
  }

  .page {
    width: 100%;
    max-width: 1100px;
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
    grid-template-columns: 90px 1fr 1fr 1fr 100px 68px 32px;
    gap: 0.75rem 1.25rem;
    align-items: end;
    padding: 0.75rem 0;
  }

  .field--bullets input {
    text-align: center;
    padding: 0 0.4rem;
  }

  .row-edu {
    padding: 0.75rem 0;
  }

  .row-edu-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 0.6rem;
  }

  .edu-fields {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.75rem 1.25rem;
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

  select {
    height: 40px;
    border: 1px solid #d1d5db;
    border-radius: 8px;
    padding: 0 0.6rem;
    font-size: 0.9rem;
    color: #111827;
    background: #f9fafb;
    transition: border-color 0.18s, box-shadow 0.18s, background 0.18s;
    outline: none;
    width: 100%;
    box-sizing: border-box;
    cursor: pointer;
    appearance: auto;
  }

  select:focus {
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
    line-height: 1.5;
  }

  textarea::placeholder {
    color: #9ca3af;
  }

  textarea:focus {
    border-color: #4f46e5;
    background: #fff;
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.12);
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

  .generate-row {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    gap: 0.75rem;
    margin-top: 1.25rem;
  }

  .btn-generate {
    height: 42px;
    padding: 0 2rem;
    border-radius: 8px;
    font-size: 0.92rem;
    font-weight: 600;
    color: #fff;
    background: #4f46e5;
    border: 1px solid #4338ca;
    cursor: pointer;
    transition: background 0.18s, box-shadow 0.18s;
    letter-spacing: 0.01em;
  }

  .btn-generate:hover {
    background: #4338ca;
    box-shadow: 0 2px 8px rgba(79, 70, 229, 0.28);
  }

  .btn-generate:active {
    background: #3730a3;
  }

  .btn-generate:disabled {
    opacity: 0.6;
    cursor: not-allowed;
    box-shadow: none;
  }

  /* Eligibility alert banner */
  .alert-banner {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 100;
    display: flex;
    align-items: center;
    gap: 0.75rem;
    padding: 0.85rem 1.5rem;
    font-size: 0.9rem;
    font-weight: 500;
    animation: slideDown 0.25s ease;
  }

  @keyframes slideDown {
    from { transform: translateY(-100%); opacity: 0; }
    to   { transform: translateY(0);     opacity: 1; }
  }

  .alert-eligible {
    background: #f0fdf4;
    border-bottom: 1px solid #bbf7d0;
    color: #166534;
  }

  .alert-ineligible {
    background: #fff7ed;
    border-bottom: 1px solid #fed7aa;
    color: #9a3412;
  }

  .alert-icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 50%;
    font-size: 0.75rem;
    font-weight: 700;
    flex-shrink: 0;
  }

  .alert-eligible .alert-icon {
    background: #dcfce7;
    color: #16a34a;
  }

  .alert-ineligible .alert-icon {
    background: #ffedd5;
    color: #ea580c;
  }

  .alert-text {
    flex: 1;
  }

  .alert-close {
    height: auto;
    padding: 0.1rem 0.4rem;
    border-radius: 4px;
    font-size: 1.1rem;
    font-weight: 400;
    line-height: 1;
    background: transparent;
    border: none;
    cursor: pointer;
    opacity: 0.5;
    transition: opacity 0.15s;
    flex-shrink: 0;
  }

  .alert-close:hover {
    opacity: 1;
  }

  @media (max-width: 560px) {
    .form-grid {
      grid-template-columns: 1fr;
    }

    .row-grid {
      grid-template-columns: 1fr 1fr 1fr 68px 32px;
    }

    .row-label {
      grid-column: 1 / -1;
      padding-bottom: 0;
    }

    .edu-fields {
      grid-template-columns: 1fr;
    }
  }
</style>

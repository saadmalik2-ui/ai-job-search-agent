# Integration notes

This is a local prototype. Configure credentials and identifiers in your own n8n instance rather than copying values from someone else's environment.

## Services

| Service | Example local endpoint |
| --- | --- |
| n8n | `http://localhost:5678` |
| Ollama | `http://host.docker.internal:11434` from an n8n Docker container on macOS |
| SearXNG | `http://host.docker.internal:8080` from an n8n Docker container on macOS |

Allow only authorized local access. Enable JSON output in SearXNG and verify that configured upstream search providers return results. Do not bypass provider access restrictions.

## Data tables

Create a private `Company Contact Directory` table with string columns `CompanyKey`, `ContactName`, `ContactTitle`, `SourceURL`, and `Evidence`. `CompanyKey` is the normalized company identifier (lowercase, alphanumeric). Populate only with independently sourced professional evidence.

Create a private `Job Search Results` table with string columns `Company`, `JobTitle`, `ContactName`, `ContactTitle`, `SourceURL`, `OutreachDraft`, `OutreachStatus`, and `ApplicationStatus`; use a number column `FitScore`. The dashboard should not display private application metadata.

## Form fields

- Company Name (required)
- job_title (required)
- Job Description (required)
- contact_focus (optional)
- job_url (optional)
- resume (required PDF)

## Verification checklist

1. Submit a synthetic job posting and résumé. Verify extracted text and company name.
2. Inspect the three fit-score category values and ensure the displayed sum is valid.
3. Verify every contact has a name, role and source. Reject unsupported claims.
4. Simulate a search provider outage using a safe fixture. Confirm directory fallback produces draft-only results for an existing known company.
5. Confirm a company with no evidence returns no contact and no draft.
6. Check that repeated executions do not create duplicated contact cards and that all outreach remains pending review.

## What is not included

Production-grade verification, a universal contact database, automated messaging, private job status scraping, and an automatic hiring decision system are not part of this prototype.

## Implementation extracts

These are sanitized Code-node extracts from the working prototype, with personal score overrides removed and candidate names replaced. They demonstrate the logic but are not a complete importable n8n workflow. The score calculation expects the `analysis` string from the fit assessment; the other extracts depend on the named n8n nodes and data table schema. Validate sources independently before using any recovered professional data.

### Score calculation

```js
const clean=analysis.replace(/\*\*/g,'').replace(/\\n/g,'\n');
const scorePart=(label,cap)=>{const hit=clean.match(new RegExp('^\\s*(?:[-*]\\s*)?'+label+'\\s*\\('+cap+'\\s*(?:pts|points)\\)\\s*:\\s*(\\d{1,3})\\s*(?:pts|points)?','im')); return hit&&Number(hit[1])<=cap?Number(hit[1]):null;};
const parts=[scorePart('Must[- ]haves?',70),scorePart('Core responsibilities',20),scorePart('Preferred(?: qualifications)?',10)];
const fit_score=parts.every(x=>x!==null)?parts.reduce((a,b)=>a+b,0):null;

```

### Source validation and review-only drafts

```js
const research = $("Call 'AI Job Research Agent'").first().json.output || {};
const contacts = Array.isArray(research.contact_sources) ? research.contact_sources : [];
const companySources = Array.isArray(research.company_sources) ? research.company_sources : [];
const text=String($input.first().json.text||'').replace(/\*\*/g,'').replace(/\\n/g,'\n');
const normalize=s=>String(s||'').toLowerCase().replace(/\s+/g,' ').trim();
const host=u=>(String(u).match(/^https?:\/\/([^/?#]+)/i)?.[1]||'').toLowerCase().replace(/^www\./,'');
const companyKey=normalize(research.company).replace(/[^a-z0-9]/g,'');
const rolePattern=/sales|revenue|account executive|account management|customer success|solutions|presales|pre-sales|recruit|talent acquisition|people|hiring team|contact person/i;
const field=(block,key)=>block.match(new RegExp('^\\s*'+key+':\\s*(.+)$','im'))?.[1]?.trim()||'';
const allowed=(url)=>{const d=host(url);return (/(^|\.)linkedin\.com$/.test(d)&&/\/in\//.test(url))||(companyKey.length>=3&&d.replace(/[^a-z0-9]/g,'').includes(companyKey)&&companySources.some(s=>host(s.url)===d))||(['join.com','jobs.ashbyhq.com','boards.greenhouse.io','job-boards.greenhouse.io','jobs.lever.co','jobs.workable.com','apply.workable.com'].includes(d)&&/https:\/\//i.test(url));};
const directoryDomain=r=>['signalhire.com','theorg.com','crunchbase.com','zoominfo.com','rocketreach.co'].some(d=>host(r.url)===d||host(r.url).endsWith('.'+d));
const isProfile=r=>/(^|\.)linkedin\.com$/.test(host(r.url))&&/\/in\//.test(r.url);
const textOf=r=>normalize(String(r.title||'')+' '+String(r.content||''));
const matchesIdentity=(r,name)=>textOf(r).includes(normalize(name))&&textOf(r).includes(normalize(research.company));
const roleNear=(r,name,role)=>{const s=textOf(r), n=normalize(name), t=normalize(role), i=s.indexOf(n); if(i<0||!t)return false;const window=s.slice(Math.max(0,i-80),i+n.length+100); return window.includes(t);};
const pair=(name,role,source)=>isProfile(source)?contacts.find(r=>host(r.url)!==host(source.url)&&directoryDomain(r)&&matchesIdentity(r,name)&&roleNear(r,name,role)):null;
const executiveRole=/\b(?:chief revenue officer|CRO|(?:vice president|VP|head|director|manager) of (?:sales engineering|solutions engineering|sales|revenue|customer success|account management|talent acquisition)|(?:sales engineering|solutions engineering|talent acquisition|sales|recruiting|recruitment|customer success|account management) (?:director|manager|lead|specialist)|recruiter|hiring team)\b/i;
const drafts=[]; let excluded=0;
for(const block of text.split(/(?=FOR REVIEW)/i).filter(b=>/Recipient:/i.test(b))){
 const name=field(block,'Recipient'), role=field(block,'Role'), url=field(block,'Source').match(/https?:\/\/[^\s)>\]]+/i)?.[0]||'', subject=field(block,'Subject'), body=block.match(/^\s*Draft:\s*([\s\S]+)$/im)?.[1]?.trim()||'';
 const source=contacts.find(r=>r.url===url), evidence=normalize(source?source.title+' '+source.content:'');
 const second=source?pair(name,role,source):null;
 if(!name||!role||!url||!subject||!body||!source||!allowed(url)||!evidence.includes(normalize(name))||!(evidence.includes(normalize(role))||second)||!evidence.includes(normalize(research.company))||!rolePattern.test(role)||body.split(/\s+/).length>120||drafts.some(d=>normalize(d.recipient)===normalize(name))){excluded++;continue;}
 drafts.push({recipient:name,role:second&&!evidence.includes(normalize(role))?role+' (directory-reported; verify)':role,source_url:url,source_title:source.title,source_excerpt:source.content+(second?' | Title corroboration: '+second.url+' '+second.content:''),verification_status:second?'CROSS_SOURCE; CONFIRM_CURRENT_ROLE':'MATCHED_TO_SEARCH_SOURCE; CONFIRM_CURRENT_ROLE',subject,draft:body,approved:false});
}
const jd=$('Edit Fields').first().json; const job=String(jd.job_title||'the advertised role');
for(const source of contacts.filter(isProfile)){if(drafts.length>=5)break;
 const name=String(source.title||'').split(/\s+[–—|-]\s+/)[0].trim();
 if(!/^[\p{Lu}][\p{L}'’.-]+(?:[ -][\p{Lu}][\p{L}'’.-]+){1,4}$/u.test(name)||!matchesIdentity(source,name)||drafts.some(d=>normalize(d.recipient)===normalize(name)))continue;
 const second=contacts.find(r=>directoryDomain(r)&&matchesIdentity(r,name)&&host(r.url)!==host(source.url)&&(()=>{const s=String(r.title||'')+' '+String(r.content||''), i=s.toLowerCase().indexOf(name.toLowerCase());return i>=0&&executiveRole.test(s.slice(i+name.length,i+name.length+100));})());
 if(!second)continue; const s=String(second.title||'')+' '+String(second.content||''), i=s.toLowerCase().indexOf(name.toLowerCase()); const role=s.slice(i+name.length,i+name.length+100).match(executiveRole)?.[0]; if(!role)continue;
 const subject=job+' opportunity at '+research.company;
 const body='Hi '+name.split(' ')[0]+',\n\nI came across the '+job+' opportunity at '+research.company+' and wanted to connect. My background is in customer-facing technical sales and solutions engineering. I would welcome the chance to learn more about the team and how my experience could contribute. Would you be open to a brief conversation?\n\nBest,\nCandidate';
 drafts.push({recipient:name,role:role+' (directory-reported; verify)',source_url:source.url,source_title:source.title,source_excerpt:source.content+' | Title corroboration: '+second.url+' '+second.content,verification_status:'CROSS_SOURCE; CONFIRM_CURRENT_ROLE',subject,draft:body,approved:false});
}
return [{json:{company:research.company,job_fit_analysis:$('Basic LLM Chain').first().json.text||'',status:drafts.length?'DRAFTS_FOR_REVIEW':'NO_SOURCE_BACKED_DRAFT',approval_required:true,outreach_sent:false,drafts,excluded_drafts:excluded,company_sources:companySources,contact_sources:contacts,search_errors:research.contact_search_errors||[]}}];
```

### Dashboard rendering

```js
const esc = v => String(v ?? '').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
const raw = $input.all().map(x=>x.json).filter(x=>x.Company || x.JobTitle);
const jobKey=r=>String(r.Company||'').trim().toLowerCase()+'|'+String(r.JobTitle||'').trim().toLowerCase();
const resolved=new Set(raw.filter(r=>String(r.ContactName||'').trim()).map(jobKey));
const manualScores={};
const override=r=>manualScores[String(r.Company||'').toLowerCase().replace(/[^a-z0-9]/g,'')+'|'+String(r.JobTitle||'').trim().toLowerCase()];
const rank=r=>typeof r.FitScore==='number'&&r.FitScore>=0&&r.FitScore<=100?r.FitScore:-1;
const rows=raw.filter(r=>String(r.ContactName||'').trim()||!resolved.has(jobKey(r))).map(r=>override(r)!==undefined?{...r,FitScore:override(r),manualScore:true}:r).sort((a,b)=>rank(b)-rank(a)||String(a.Company||'').localeCompare(String(b.Company||''))||String(a.JobTitle||'').localeCompare(String(b.JobTitle||'')));
const cards = rows.map((r,i)=>{
 const score=typeof r.FitScore==='number'&&r.FitScore>=0&&r.FitScore<=100?r.FitScore+'/100':'Not scored';
 const scoreClass=score==='Not scored'?'unscored':r.FitScore>=80?'high':r.FitScore>=60?'medium':'low';
 const url=String(r.SourceURL||'');
 const link=/^https?:\/\/[^\s]+$/i.test(url)?'<a class="source" href="'+esc(url)+'" target="_blank" rel="noopener noreferrer">View source ↗</a>':'<span class="muted">No source yet</span>';
 const draft=String(r.OutreachDraft||'');
 return '<article><div class="top"><div><div class="company">'+esc(r.Company)+'</div><h2>'+esc(r.JobTitle)+'</h2></div><span class="score '+scoreClass+'">'+esc(score)+'</span></div><div class="contact"><strong>'+esc(r.ContactName||'No contact identified')+'</strong><span>'+esc(r.ContactTitle||'')+'</span>'+link+'</div><div class="status">'+esc(r.OutreachStatus||'Pending review')+'</div><div class="drafthead"><h3>Outreach draft</h3>'+(draft?'<button type="button" data-copy="draft-'+i+'">Copy draft</button>':'')+'</div><textarea id="draft-'+i+'" readonly rows="8" aria-label="Outreach draft">'+esc(draft||'No draft generated. Review research evidence and try again later.')+'</textarea></article>';
}).join('');
const html='<!doctype html><html lang="en"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1"><title>Job Search Results</title><style>*{box-sizing:border-box}body{margin:0;background:#f4f6f9;color:#162332;font:16px system-ui,sans-serif}main{max-width:1040px;margin:0 auto;padding:36px 20px}.eyebrow{color:#176b64;letter-spacing:.12em;text-transform:uppercase;font-size:12px;font-weight:750}h1{margin:8px 0;font-size:34px;letter-spacing:-.04em}header p{color:#5b6875;margin:8px 0 28px}.count{font-weight:700;color:#176b64}.grid{display:grid;gap:18px}article{background:white;border:1px solid #dce3e9;border-radius:16px;padding:24px;box-shadow:0 3px 12px #1423320b}.top{display:flex;align-items:flex-start;justify-content:space-between;gap:20px}.company{font-size:14px;color:#176b64;font-weight:800;text-transform:uppercase;letter-spacing:.06em}h2{font-size:23px;margin:5px 0 20px}.score{background:#e1f4ed;color:#11634f;border-radius:999px;padding:10px 15px;font-weight:800;white-space:nowrap}.score.medium{background:#fff1d6;color:#9a5300}.score.low{background:#ffe2e5;color:#a11b2a}.score.unscored{background:#eceff3;color:#576374}.contact{display:flex;gap:12px;flex-wrap:wrap;align-items:center;color:#526273}.contact strong{color:#162332}.source{color:#176b64;font-weight:700}.status{margin-top:14px;display:inline-block;border-radius:8px;background:#fff2d8;color:#80590b;padding:6px 10px;font-size:13px;font-weight:700}.drafthead{display:flex;justify-content:space-between;align-items:center;margin-top:18px}h3{font-size:16px;margin:0}button{border:1px solid #176b64;border-radius:9px;background:#176b64;color:#fff;font:inherit;font-size:14px;padding:9px 15px;cursor:pointer}textarea{display:block;width:100%;margin-top:12px;background:#f7f9fb;border:1px solid #dce3e9;border-radius:10px;padding:16px;color:#162332;font:15px/1.65 system-ui,sans-serif;resize:vertical;min-height:180px;field-sizing:content}.muted{color:#6a7682}.foot{margin-top:24px;color:#6a7682;font-size:13px}@media(max-width:560px){main{padding:20px 12px}article{padding:16px}h1{font-size:28px}h2{font-size:19px}}</style></head><body><main><header><div class="eyebrow">Local job search assistant</div><h1>Job Search Results</h1><p><span class="count">'+rows.length+(rows.length===1?' result':' results')+'</span> · Sorted highest fit score first. Review contacts and outreach drafts. Nothing is sent automatically.</p><p>Fit: <span class="score high">80–100</span> <span class="score medium">60–79</span> <span class="score low">0–59</span></p></header><section class="grid">'+(cards||'<article>No results yet. Run the job search workflow first.</article>')+'</section><p class="foot">Source links are evidence for human review. Fit scores are AI estimates unless adjusted by you. User-adjusted scores are retained across runs.</p></main><script>document.querySelectorAll("[data-copy]").forEach(b=>b.addEventListener("click",async()=>{const t=document.getElementById(b.dataset.copy);t.focus();t.select();let ok=false;try{if(navigator.clipboard&&navigator.clipboard.writeText){await navigator.clipboard.writeText(t.value);ok=true;}}catch(e){}if(!ok){try{ok=document.execCommand("copy")}catch(e){}}b.textContent=ok?"Copied":"Selected — press Cmd/Ctrl+C";setTimeout(()=>b.textContent="Copy draft",2400)}));</script></body></html>';
return [{json:{html}}];
```

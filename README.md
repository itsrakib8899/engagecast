import axios from 'axios';

const PRIVY_APP_SECRET = process.env.PRIVY_APP_SECRET;

// Placeholder verify function: adjust to Privy docs as needed.
export async function verifyPrivyToken(token: string) {
  // Privy verification endpoint — replace based on Privy docs if needed.
  // This code expects Privy to accept a token and return user profile:
  // { fid, username, display_name, pfp }
  const resp = await axios.post('https://api.privy.io/v1/auth/verify', { token }, {
    headers: { Authorization: `Bearer ${PRIVY_APP_SECRET}` }
  });
  return resp.data;
}

import { createClient } from '@supabase/supabase-js';

const SUPABASE_URL = process.env.SUPABASE_URL!;
const SUPABASE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY!;

export const supabaseAdmin = createClient(SUPABASE_URL, SUPABASE_KEY);

export const supabaseClient = createClient(
  SUPABASE_URL,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
[index.tsx](https://github.com/user-attachments/files/23140259/index.tsx)
import React, { useEffect, useState } from 'react';
import axios from 'axios';

export default function Home() {
  const [jobs, setJobs] = useState([]);
  const [user, setUser] = useState<any>(null);

  useEffect(() => { fetchJobs(); }, []);

  async function fetchJobs() {
    const res = await fetch('/api/jobs');
    const json = await res.json();
    setJobs(json.jobs || []);
  }

  async function signIn() {
    // For dev: ask user to paste a Privy token from the browser workflow
    const token = prompt('Paste Privy token (dev) — for production integrate PrivyJS flow');
    if (!token) return;
    const resp = await axios.post('/api/auth', { token });
    setUser(resp.data.user);
  }

  async function createJob() {
    if (!user) return alert('Please sign in');
    const engagement_type = prompt('Engagement type (comment/follow/recast/like)') || 'comment';
    const post_link = prompt('Farcaster post link') || '';
    const reward_points = Number(prompt('Reward points') || '10');
    await axios.post('/api/jobs', { creator_fid: user.fid, engagement_type, post_link, reward_points });
    fetchJobs();
  }

  return (
    <div style={{ padding: 24, fontFamily: 'Arial, sans-serif' }}>
      <header style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 20 }}>
        <h1>EngageCast</h1>
        <div>
          {user ? <span style={{ marginRight: 12 }}>{user.display_name} ({user.points} pts)</span> : null}
          <button onClick={signIn}>{user ? 'Refresh' : 'Sign in with Farcaster (Privy)'}</button>
        </div>
      </header>

      <section style={{ marginBottom: 20 }}>
        <button onClick={createJob}>Create Job</button>
      </section>

      <section>
        <h2>Marketplace</h2>
        <div>
          {jobs.map(j => (
            <div key={j.id} style={{ border: '1px solid #ddd', padding: 12, marginBottom: 8 }}>
              <div style={{ display: 'flex', justifyContent: 'space-between' }}>
                <div>
                  <div style={{ fontWeight: 600 }}>{j.engagement_type}</div>
                  <a href={j.post_link} target="_blank" rel="noreferrer">Open post</a>
                </div>
                <div style={{ textAlign: 'right' }}>
                  <div style={{ fontWeight: 700 }}>{j.reward_points} pts</div>
                  <div style={{ fontSize: 12 }}>Posted: {new Date(j.created_at).toLocaleString()}</div>
                </div>
              </div>
            </div>
          ))}
        </div>
      </section>
    </div>
  );
}

import type { NextApiRequest, NextApiResponse } from 'next';
import { verifyPrivyToken } from '../../lib/privy';
import { supabaseAdmin } from '../../lib/supabaseClient';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') return res.status(405).end();
  const { token } = req.body;
  if (!token) return res.status(400).json({ error: 'missing token' });

  try {
    const profile = await verifyPrivyToken(token);
    // profile should contain { fid, username, display_name, pfp }
    const fid = profile.fid;
    // Auto create user if not exists. Start points from env default.
    const startPoints = Number(process.env.NEXT_PUBLIC_START_POINTS || '0');

    const { data, error } = await supabaseAdmin
      .from('users')
      .upsert({ fid, username: profile.username, display_name: profile.display_name, pfp: profile.pfp, points: startPoints }, { onConflict: 'fid' })
      .select('*')
      .single();

    if (error) throw error;
    return res.json({ user: data });
  } catch (err: any) {
    console.error('auth error', err.message || err);
    return res.status(500).json({ error: 'auth_failed' });
  }
}

import type { NextApiRequest, NextApiResponse } from 'next';
import { supabaseAdmin } from '../../lib/supabaseClient';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    const { data, error } = await supabaseAdmin.from('jobs').select('*').order('created_at', { ascending: false });
    if (error) return res.status(500).json({ error });
    return res.json({ jobs: data });
  }

  if (req.method === 'POST') {
    const { creator_fid, engagement_type, post_link, reward_points } = req.body;
    if (!creator_fid || !engagement_type || !post_link || !reward_points) return res.status(400).json({ error: 'missing_fields' });

    // check creator exists and has enough points
    const { data: creator } = await supabaseAdmin.from('users').select('*').eq('fid', creator_fid).single();
    if (!creator) return res.status(400).json({ error: 'creator_not_found' });
    if (creator.points < reward_points) return res.status(400).json({ error: 'insufficient_points' });

    const { data, error } = await supabaseAdmin.from('jobs').insert([{ creator_fid, engagement_type, post_link, reward_points }]).select('*').single();
    if (error) return res.status(500).json({ error });
    return res.json({ job: data });
  }

  return res.status(405).end();
}

import type { NextApiRequest, NextApiResponse } from 'next';
import { supabaseAdmin } from '../../lib/supabaseClient';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') return res.status(405).end();
  const { job_id, worker_fid, proof_link } = req.body;
  if (!job_id || !worker_fid || !proof_link) return res.status(400).json({ error: 'missing_fields' });

  // duplicate check
  const { data: dup } = await supabaseAdmin.from('submissions').select('*').eq('job_id', job_id).eq('worker_fid', worker_fid);
  if (dup && dup.length > 0) return res.status(400).json({ error: 'duplicate_submission' });

  const { data, error } = await supabaseAdmin.from('submissions').insert([{ job_id, worker_fid, proof_link }]).select('*').single();
  if (error) return res.status(500).json({ error });
  return res.json({ submission: data });
}

import type { NextApiRequest, NextApiResponse } from 'next';
import { supabaseAdmin } from '../../lib/supabaseClient';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') return res.status(405).end();
  const { submission_id, approver_fid, action } = req.body; // action: approve / reject
  if (!submission_id || !approver_fid || !action) return res.status(400).json({ error: 'missing_fields' });

  const { data: submission } = await supabaseAdmin.from('submissions').select('*').eq('id', submission_id).single();
  if (!submission) return res.status(404).json({ error: 'submission_not_found' });

  const { data: job } = await supabaseAdmin.from('jobs').select('*').eq('id', submission.job_id).single();
  if (!job) return res.status(404).json({ error: 'job_not_found' });
  if (String(job.creator_fid) !== String(approver_fid)) return res.status(403).json({ error: 'not_job_owner' });

  if (action === 'approve') {
    const points = job.reward_points;
    const { data: creator } = await supabaseAdmin.from('users').select('*').eq('fid', job.creator_fid).single();
    const { data: worker } = await supabaseAdmin.from('users').select('*').eq('fid', submission.worker_fid).single();
    if (!creator || !worker) return res.status(500).json({ error: 'user_missing' });
    if (creator.points < points) return res.status(400).json({ error: 'insufficient_points_creator' });

    await supabaseAdmin.from('users').update({ points: creator.points - points }).eq('fid', creator.fid);
    await supabaseAdmin.from('users').update({ points: worker.points + points }).eq('fid', worker.fid);

    await supabaseAdmin.from('submissions').update({ status: 'approved' }).eq('id', submission_id);
    await supabaseAdmin.from('transactions').insert([{ from_fid: creator.fid, to_fid: worker.fid, points, job_id: job.id }]);

    return res.json({ ok: true });
  }

  if (action === 'reject') {
    await supabaseAdmin.from('submissions').update({ status: 'rejected' }).eq('id', submission_id);
    return res.json({ ok: true });
  }

  return res.status(400).json({ error: 'invalid_action' });
}
[schema.sql](https://github.com/user-attachments/files/23140310/schema.sql)
-- Users Table (Farcaster users)
create extension if not exists pgcrypto;

create table if not exists users (
  id uuid primary key default gen_random_uuid(),
  fid bigint unique not null,
  username text,
  display_name text,
  pfp text,
  points int default 0,
  created_at timestamp default now()
);

-- Jobs Table (tasks posted by users)
create table if not exists jobs (
  id uuid primary key default gen_random_uuid(),
  creator_fid bigint not null,
  engagement_type text not null,
  post_link text not null,
  reward_points int not null,
  status text default 'open',
  created_at timestamp default now()
);

-- Submissions (proof of completion)
create table if not exists submissions (
  id uuid primary key default gen_random_uuid(),
  job_id uuid references jobs(id),
  worker_fid bigint not null,
  proof_link text,
  status text default 'pending',
  created_at timestamp default now()
);

-- Transactions (point transfer logs)
create table if not exists transactions (
  id uuid primary key default gen_random_uuid(),
  from_fid bigint,
  to_fid bigint,
  points int,
  job_id uuid,
  created_at timestamp default now()
);
[README.md](https://github.com/user-attachments/files/23140353/README.md)
[package.json](https://github.com/user-attachments/files/23140352/package.json)




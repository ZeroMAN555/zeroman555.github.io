---
layout: page
title: Achievements
icon: fas fa-trophy
order: 2
---
<!-- 1 -->
<div class="achievement-gallery">
  <div class="achievement-card">
    <div class="card-image">
      <img src="/assets/img/certificates/cert1.jpg" alt="Belajar Coding" loading="lazy">
      <div class="card-overlay">
      </div>
    </div>
    <div class="card-content">
      <h3>Basic Web Security Pentester & Hacking</h3>
      <p>SIBERMUDA ID</p>
      <p>August 2024</p>
    </div>
  </div>

  <!-- 2 -->
  <div class="achievement-card">
    <div class="card-image">
      <img src="/assets/img/certificates/cert2.jpg" alt="CTF Challenge" loading="lazy">
      <div class="card-overlay">
      </div>
    </div>
    <div class="card-content">
      <h3>Certificate of Appreciation</h3>
      <p>BangkalanKab-CSIRT</p>
      <p>March 2025</p>
    </div>
  </div>

  <!-- 3 -->
  <div class="achievement-card">
    <div class="card-image">
      <img src="/assets/img/certificates/cert3.jpg" alt="Achievement Unlock" loading="lazy">
      <div class="card-overlay">
      </div>
    </div>
    <div class="card-content">
      <h3>Certificate of Appreciation</h3>
      <p>Diskominfotik Provinsi DKI Jakarta</p>
      <p>March 2025</p>
    </div>
  </div>
  <!-- 4 -->
  <div class="achievement-card">
    <div class="card-image">
      <img src="/assets/img/certificates/cert4.jpg" alt="Web Security" loading="lazy">
      <div class="card-overlay">
      </div>
    </div>
    <div class="card-content">
      <h3>Introduction to Information Security Course</h3>
      <p>Cyber Academy</p>
      <p>May 2025</p>
    </div>
  </div>
  <!-- 5 -->
  <div class="achievement-card">
    <div class="card-image">
      <img src="/assets/img/certificates/cert5.jpg" alt="GitHub Project" loading="lazy">
      <div class="card-overlay">
      </div>
    </div>
    <div class="card-content">
      <h3>Ethical Hacker Course</h3>
      <p>Cisco Networking Academy</p>
      <p>September 2025</p>
    </div>
  </div>

</div>

> **"Setiap achievement dimulai dari keputusan untuk mencoba."** - Catatan perjalanan belajar yang terus berlanjut.

<style>
/* Achievement Gallery Styles */
.achievement-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
  gap: 2rem;
  margin: 2rem 0;
}

.achievement-card {
  background: var(--card-bg, #ffffff);
  border-radius: 15px;
  overflow: hidden;
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  border: 1px solid var(--card-border-color, rgba(0, 0, 0, 0.1));
}

.achievement-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.15);
}

.card-image {
  position: relative;
  overflow: hidden;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.card-image img {
  max-width: 100%;
  max-height: 100%;
  width: auto;
  height: auto;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.achievement-card:hover .card-image img {
  transform: scale(1.05);
}

.card-overlay {
  position: absolute;
  top: 15px;
  left: 15px;
}

.card-category {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 20px;
  font-size: 0.8rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.card-content {
  padding: 1.5rem;
}

.card-content h3 {
  margin: 0 0 0.75rem 0;
  color: var(--heading-color, #2d3748);
  font-size: 1.25rem;
  font-weight: 700;
}

.card-content p {
  margin: 0 0 1rem 0;
  color: var(--text-muted, #718096);
  line-height: 1.6;
  font-size: 0.95rem;
}



/* Stats Container */
.stats-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin: 2rem 0;
  padding: 2rem;
  background: var(--code-bg, #f8f9fa);
  border-radius: 15px;
  border-left: 5px solid var(--link-color, #0969da);
}

.stat-item {
  text-align: center;
}

.stat-number {
  font-size: 2.5rem;
  font-weight: 800;
  color: var(--link-color, #0969da);
  line-height: 1;
  margin-bottom: 0.5rem;
}

.stat-label {
  font-size: 0.9rem;
  color: var(--text-muted, #718096);
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Dark mode support */
[data-mode="dark"] .achievement-card {
  background: var(--card-bg);
  border-color: var(--card-border-color);
}

[data-mode="dark"] .card-content h3 {
  color: var(--heading-color);
}

[data-mode="dark"] .card-content p {
  color: var(--text-muted);
}



[data-mode="dark"] .stats-container {
  background: var(--code-bg);
}

/* Responsive Design */
@media (max-width: 768px) {
  .achievement-gallery {
    grid-template-columns: 1fr;
    gap: 1.5rem;
  }
  
  .stats-container {
    grid-template-columns: repeat(2, 1fr);
    padding: 1.5rem;
  }
  
  .stat-number {
    font-size: 2rem;
  }
}

@media (max-width: 480px) {
  .stats-container {
    grid-template-columns: 1fr;
  }
}
</style>
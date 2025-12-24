using System;
using System.Collections;
using System.Collections.Generic;
using System.Reflection;
using BepInEx;
using ExitGames.Client.Photon;
using GorillaLocomotion;
using Photon.Pun;
using Photon.Realtime;
using TMPro;
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;

namespace Checker
{
	// Token: 0x02000008 RID: 8
	[BepInPlugin("Creator.VaelCatcher.Checker", "Checker", "1.0.2")]
	public class Checker : BaseUnityPlugin
	{
		// Token: 0x0600001F RID: 31 RVA: 0x00002EF8 File Offset: 0x000010F8
		private void Awake()
		{
			this.ApplyTheme(1);
			bool flag = this.publishPingRoutine == null;
			if (flag)
			{
				this.publishPingRoutine = base.StartCoroutine(this.PublishLocalPingRoutine());
			}
			base.StartCoroutine(this.DownloadAllIcons());
		}

		// Token: 0x06000020 RID: 32 RVA: 0x00002F3A File Offset: 0x0000113A
		private IEnumerator DownloadAllIcons()
		{
			foreach (KeyValuePair<string, string> kvp in this.iconUrls)
			{
				yield return base.StartCoroutine(this.DownloadIcon(kvp.Key, kvp.Value));
				kvp = default(KeyValuePair<string, string>);
			}
			Dictionary<string, string>.Enumerator enumerator = default(Dictionary<string, string>.Enumerator);
			this.UpdateSidebarIcons();
			yield break;
			yield break;
		}

		// Token: 0x06000021 RID: 33 RVA: 0x00002F49 File Offset: 0x00001149
		private IEnumerator DownloadIcon(string key, string url)
		{
			using (UnityWebRequest request = UnityWebRequestTexture.GetTexture(url))
			{
				yield return request.SendWebRequest();
				bool flag = request.result == UnityWebRequest.Result.Success;
				if (flag)
				{
					Texture2D texture = DownloadHandlerTexture.GetContent(request);
					Sprite sprite = Sprite.Create(texture, new Rect(0f, 0f, (float)texture.width, (float)texture.height), new Vector2(0.5f, 0.5f));
					this.downloadedIcons[key] = sprite;
					texture = null;
					sprite = null;
				}
			}
			UnityWebRequest request = null;
			yield break;
			yield break;
		}

		// Token: 0x06000022 RID: 34 RVA: 0x00002F68 File Offset: 0x00001168
		private void UpdateSidebarIcons()
		{
			int num = 0;
			while (num < this.sidebarIconImages.Count && num < this.pageIconKeys.Length)
			{
				bool flag = this.sidebarIconImages[num] != null && this.downloadedIcons.ContainsKey(this.pageIconKeys[num]);
				if (flag)
				{
					this.sidebarIconImages[num].sprite = this.downloadedIcons[this.pageIconKeys[num]];
					this.sidebarIconImages[num].color = Color.white;
				}
				num++;
			}
		}

		// Token: 0x06000023 RID: 35 RVA: 0x00003010 File Offset: 0x00001210
		private void Update()
		{
			bool flag = ControllerInputPoller.instance == null;
			if (!flag)
			{
				bool leftY = SimpleInputs.LeftY;
				bool flag2 = leftY && !this.lastXButtonState;
				if (flag2)
				{
					bool flag3 = this.uiPanel == null;
					if (flag3)
					{
						this.SpawnUIPanel();
					}
					else
					{
						this.uiPanel.SetActive(true);
					}
					bool flag4 = this.fadeRoutine != null;
					if (flag4)
					{
						base.StopCoroutine(this.fadeRoutine);
					}
					this.SetPanelAlpha(1f);
					this.ShowPanel(true);
					this.StartCreationDateRefresh();
				}
				bool flag5 = !leftY && this.lastXButtonState;
				if (flag5)
				{
					this.FadeOutPanel();
					this.StopCreationDateRefresh();
				}
				this.lastXButtonState = leftY;
				this.UpdateUIPanelPosition();
				this.UpdateGunRay();
				this.UpdateLobbyInfo();
				this.CheckButtonTouch();
				VRRig vrrig = this.selectedRig ?? this.speedSampleRig;
				bool flag6 = this.selectedRig != null;
				if (flag6)
				{
					this.playerFPSText.text = "<b>FPS:</b> " + GTUtility.GetFPS(this.selectedRig);
					this.playerCosmeticsText.text = "<b>Cosmetics:</b> " + GTUtility.GetCosmetics(this.selectedRig);
					int playerPing = this.GetPlayerPing(this.selectedRig);
					this.playerPingText.text = "<b>Ping:</b> " + this.GetPingDisplay(playerPing);
					bool flag7 = this.playerColorCodeText != null;
					if (flag7)
					{
						this.playerColorCodeText.GetComponent<TextMeshProUGUI>().text = "<b>Color:</b> " + this.GetColorDigits(this.selectedRig);
					}
				}
				this.UpdateCheaterDetection(this.selectedRig, vrrig);
				this.UpdateSpeedDisplay(vrrig);
			}
		}

		// Token: 0x06000024 RID: 36 RVA: 0x000031D8 File Offset: 0x000013D8
		private void StartCreationDateRefresh()
		{
			this.StopCreationDateRefresh();
			this.creationDateRefreshRoutine = base.StartCoroutine(this.RefreshCreationDateRoutine());
		}

		// Token: 0x06000025 RID: 37 RVA: 0x000031F4 File Offset: 0x000013F4
		private void StopCreationDateRefresh()
		{
			bool flag = this.creationDateRefreshRoutine != null;
			if (flag)
			{
				base.StopCoroutine(this.creationDateRefreshRoutine);
				this.creationDateRefreshRoutine = null;
			}
		}

		// Token: 0x06000026 RID: 38 RVA: 0x00003225 File Offset: 0x00001425
		private IEnumerator RefreshCreationDateRoutine()
		{
			for (;;)
			{
				yield return new WaitForSeconds(1f);
				this.RefreshCreationDate();
			}
			yield break;
		}

		// Token: 0x06000027 RID: 39 RVA: 0x00003234 File Offset: 0x00001434
		private void RefreshCreationDate()
		{
			bool flag = this.playerCreationDateText == null || this.selectedRig == null;
			if (!flag)
			{
				VRRig targetRig = this.selectedRig;
				bool flag2 = targetRig.OwningNetPlayer == null;
				if (flag2)
				{
					this.playerCreationDateText.text = "<b>Created:</b> UNKNOWN";
				}
				else
				{
					string targetUserId = targetRig.OwningNetPlayer.UserId;
					GTUtility.GetAccountCreationDate(targetRig, delegate(object date)
					{
						bool flag3 = this.playerCreationDateText != null && this.selectedRig == targetRig && targetRig.OwningNetPlayer != null && targetRig.OwningNetPlayer.UserId == targetUserId;
						if (flag3)
						{
							this.playerCreationDateText.text = string.Format("<b>Created:</b> {0}", date);
						}
					});
				}
			}
		}

		// Token: 0x06000028 RID: 40 RVA: 0x000032D4 File Offset: 0x000014D4
		private void ApplyTheme(int themeIndex)
		{
			this.currentTheme = Mathf.Clamp(themeIndex, 0, this.themePresets.Length - 1);
			Color[] array = this.themePresets[this.currentTheme];
			this.panelColor = array[0];
			this.elementDark = array[1];
			this.textLight = array[2];
			this.separatorColor = array[3];
			this.activeGreen = array[4];
			this.accentColor = array[5];
			this.sidebarColor = array[6];
			this.sidebarBtnColor = array[7];
			this.sidebarBtnActiveColor = array[8];
			this.cheaterPanelColor = array[9];
			this.cheaterPanelOutline = array[10];
			this.fadeOutColor = array[11];
		}

		// Token: 0x06000029 RID: 41 RVA: 0x000033A8 File Offset: 0x000015A8
		private void RefreshThemeUI()
		{
			bool flag = this.mainBgImage != null;
			if (flag)
			{
				this.mainBgImage.color = new Color(this.panelColor.r, this.panelColor.g, this.panelColor.b, this.menuOpacity);
			}
			bool flag2 = this.sidebarBgImage != null;
			if (flag2)
			{
				this.sidebarBgImage.color = this.sidebarColor;
			}
			for (int i = 0; i < this.sidebarButtonImages.Count; i++)
			{
				bool flag3 = this.sidebarButtonImages[i] != null;
				if (flag3)
				{
					this.sidebarButtonImages[i].color = ((i == this.currentPage - 1) ? this.sidebarBtnActiveColor : this.sidebarBtnColor);
				}
			}
			bool flag4 = this.cheaterPanelImage != null;
			if (flag4)
			{
				this.cheaterPanelImage.color = this.cheaterPanelColor;
			}
			bool flag5 = this.cheaterPanelOutlineComp != null;
			if (flag5)
			{
				this.cheaterPanelOutlineComp.effectColor = this.cheaterPanelOutline;
			}
			bool flag6 = this.gunRayToggle != null;
			if (flag6)
			{
				this.gunRayToggle.GetComponent<Image>().color = (this.gunRayEnabled ? this.activeGreen : this.elementDark);
			}
			bool flag7 = this.autoLockToggle != null;
			if (flag7)
			{
				this.autoLockToggle.GetComponent<Image>().color = (this.autoLockEnabled ? this.activeGreen : this.elementDark);
			}
			bool flag8 = this.opacityLowBtn != null;
			if (flag8)
			{
				this.opacityLowBtn.GetComponent<Image>().color = ((this.menuOpacity == 0.7f) ? this.activeGreen : this.elementDark);
			}
			bool flag9 = this.opacityMedBtn != null;
			if (flag9)
			{
				this.opacityMedBtn.GetComponent<Image>().color = ((this.menuOpacity == 0.85f) ? this.activeGreen : this.elementDark);
			}
			bool flag10 = this.opacityHighBtn != null;
			if (flag10)
			{
				this.opacityHighBtn.GetComponent<Image>().color = ((this.menuOpacity == 0.95f) ? this.activeGreen : this.elementDark);
			}
			bool flag11 = this.resetBtn != null;
			if (flag11)
			{
				this.resetBtn.GetComponent<Image>().color = this.elementDark;
			}
			this.UpdateThemeButtonColors();
			bool flag12 = this.reportBtn != null;
			if (flag12)
			{
				this.reportBtn.GetComponent<Image>().color = new Color(this.activeGreen.r * 0.8f, this.activeGreen.g * 0.4f, this.activeGreen.b * 0.4f, 1f);
			}
			bool flag13 = this.muteBtn != null;
			if (flag13)
			{
				this.muteBtn.GetComponent<Image>().color = new Color(this.activeGreen.r * 0.7f, this.activeGreen.g * 0.6f, this.activeGreen.b * 0.3f, 1f);
			}
			bool flag14 = this.blockBtn != null;
			if (flag14)
			{
				this.blockBtn.GetComponent<Image>().color = new Color(this.activeGreen.r * 0.6f, this.activeGreen.g * 0.3f, this.activeGreen.b * 0.3f, 1f);
			}
			bool flag15 = this.separatorLine != null;
			if (flag15)
			{
				this.separatorLine.GetComponent<Image>().color = this.separatorColor;
			}
			bool flag16 = this.cheaterBarBg != null;
			if (flag16)
			{
				this.cheaterBarBg.GetComponent<Image>().color = new Color(this.elementDark.r * 0.5f, this.elementDark.g * 0.5f, this.elementDark.b * 0.5f, 1f);
			}
			this.UpdateAllTextColors();
		}

		// Token: 0x0600002A RID: 42 RVA: 0x000037D4 File Offset: 0x000019D4
		private void UpdateThemeButtonColors()
		{
			for (int i = 0; i < this.themeButtons.Count; i++)
			{
				bool flag = this.themeButtons[i] != null;
				if (flag)
				{
					this.themeButtons[i].GetComponent<Image>().color = ((i == this.currentTheme) ? this.activeGreen : this.elementDark);
				}
			}
		}

		// Token: 0x0600002B RID: 43 RVA: 0x00003844 File Offset: 0x00001A44
		private void UpdateAllTextColors()
		{
			bool flag = this.titleText != null;
			if (flag)
			{
				this.titleText.color = this.textLight;
			}
			bool flag2 = this.pageText != null;
			if (flag2)
			{
				this.pageText.color = new Color(this.textLight.r * 0.65f, this.textLight.g * 0.65f, this.textLight.b * 0.65f, 1f);
			}
			bool flag3 = this.pageNameText != null;
			if (flag3)
			{
				this.pageNameText.color = this.activeGreen;
			}
			bool flag4 = this.playerNameText != null;
			if (flag4)
			{
				this.playerNameText.color = this.textLight;
			}
			bool flag5 = this.playerCreationDateText != null;
			if (flag5)
			{
				this.playerCreationDateText.color = this.textLight;
			}
			bool flag6 = this.playerModsText != null;
			if (flag6)
			{
				this.playerModsText.color = this.textLight;
			}
			bool flag7 = this.playerCosmeticsText != null;
			if (flag7)
			{
				this.playerCosmeticsText.color = this.textLight;
			}
			bool flag8 = this.importantChecksTitleText != null;
			if (flag8)
			{
				this.importantChecksTitleText.color = this.textLight;
			}
			bool flag9 = this.speedMphText != null;
			if (flag9)
			{
				this.speedMphText.color = this.textLight;
			}
			bool flag10 = this.lobbyCodeText != null;
			if (flag10)
			{
				this.lobbyCodeText.color = this.textLight;
			}
			bool flag11 = this.lobbyPlayerCountText != null;
			if (flag11)
			{
				this.lobbyPlayerCountText.color = this.textLight;
			}
			bool flag12 = this.lobbyGameModeText != null;
			if (flag12)
			{
				this.lobbyGameModeText.color = this.textLight;
			}
			bool flag13 = this.lobbyQueueText != null;
			if (flag13)
			{
				this.lobbyQueueText.color = this.textLight;
			}
			bool flag14 = this.lobbyMasterText != null;
			if (flag14)
			{
				this.lobbyMasterText.color = this.textLight;
			}
			bool flag15 = this.creditsOwnerText != null;
			if (flag15)
			{
				this.creditsOwnerText.color = this.textLight;
			}
			bool flag16 = this.creditsCoOwnerText != null;
			if (flag16)
			{
				this.creditsCoOwnerText.color = this.textLight;
			}
			bool flag17 = this.creditsHeadDevText != null;
			if (flag17)
			{
				this.creditsHeadDevText.color = this.textLight;
			}
			bool flag18 = this.creditsThankYouText != null;
			if (flag18)
			{
				this.creditsThankYouText.color = this.textLight;
			}
			bool flag19 = this.creditsText != null;
			if (flag19)
			{
				this.creditsText.color = this.textLight;
			}
			bool flag20 = this.reportStatusText != null;
			if (flag20)
			{
				this.reportStatusText.color = new Color(this.textLight.r * 0.7f, this.textLight.g * 0.7f, this.textLight.b * 0.7f, 1f);
			}
			bool flag21 = this.cheaterTitleText != null;
			if (flag21)
			{
				this.cheaterTitleText.color = new Color(1f, 0.35f, 0.35f, 1f);
			}
			bool flag22 = this.themeTitleText != null;
			if (flag22)
			{
				this.themeTitleText.color = this.textLight;
			}
		}

		// Token: 0x0600002C RID: 44 RVA: 0x00003BE4 File Offset: 0x00001DE4
		private void SetTheme(int themeIndex)
		{
			bool flag = this.uiPanel != null;
			if (flag)
			{
				base.StartCoroutine(this.ThemeTransition(themeIndex));
			}
			else
			{
				this.ApplyTheme(themeIndex);
				this.RefreshThemeUI();
			}
		}

		// Token: 0x0600002D RID: 45 RVA: 0x00003C25 File Offset: 0x00001E25
		private IEnumerator ThemeTransition(int themeIndex)
		{
			float duration = 0.15f;
			float t = 0f;
			bool flag = this.uiCanvasGroup != null;
			if (flag)
			{
				float startAlpha = this.uiCanvasGroup.alpha;
				while (t < duration)
				{
					t += Time.deltaTime;
					this.uiCanvasGroup.alpha = Mathf.Lerp(startAlpha, 0.3f, t / duration);
					yield return null;
				}
			}
			this.ApplyTheme(themeIndex);
			this.RefreshThemeUI();
			this.UpdateThemeButtonColors();
			t = 0f;
			bool flag2 = this.uiCanvasGroup != null;
			if (flag2)
			{
				while (t < duration)
				{
					t += Time.deltaTime;
					this.uiCanvasGroup.alpha = Mathf.Lerp(0.3f, 1f, t / duration);
					yield return null;
				}
				this.uiCanvasGroup.alpha = 1f;
			}
			yield break;
		}

		// Token: 0x0600002E RID: 46 RVA: 0x00003C3C File Offset: 0x00001E3C
		private float CalculateCheaterPercent(VRRig rig, VRRig speedRig)
		{
			bool flag = rig == null;
			float result;
			if (flag)
			{
				result = 0f;
			}
			else
			{
				float num = 0f;
				try
				{
					string fps = GTUtility.GetFPS(rig);
					bool flag2 = !string.IsNullOrEmpty(fps);
					if (flag2)
					{
						string text = "";
						foreach (char c in fps)
						{
							bool flag3 = char.IsDigit(c);
							if (flag3)
							{
								text += c.ToString();
							}
						}
						int num2;
						bool flag4 = !string.IsNullOrEmpty(text) && int.TryParse(text, out num2);
						if (flag4)
						{
							bool flag5 = num2 > 0 && num2 < 60;
							if (flag5)
							{
								num += 40f;
							}
						}
					}
				}
				catch
				{
				}
				try
				{
					string cosmetics = GTUtility.GetCosmetics(rig);
					bool flag6 = !string.IsNullOrEmpty(cosmetics);
					if (flag6)
					{
						string text3 = cosmetics.ToLowerInvariant();
						bool flag7 = text3.Contains("cosmeticx") || text3.Contains("cosmetic x") || text3.Contains("cosmetic_x") || text3.Contains("cx");
						if (flag7)
						{
							num += 50f;
						}
					}
				}
				catch
				{
				}
				try
				{
					string text4 = GTUtility.GetPlatform(rig);
					bool flag8 = string.IsNullOrEmpty(text4) || text4.ToLower().Contains("unknown");
					if (flag8)
					{
						string text5 = rig.concatStringOfCosmeticsAllowed ?? "";
						string text6 = text5.ToLower();
						bool flag9 = text6.Contains("steam") || text6.Contains("steamvr");
						if (flag9)
						{
							text4 = "Steam";
						}
						else
						{
							bool flag10 = text6.Contains("rift") || text6.Contains("pcvr");
							if (flag10)
							{
								text4 = "Rift";
							}
						}
					}
					bool flag11 = !string.IsNullOrEmpty(text4);
					if (flag11)
					{
						string text7 = text4.ToLowerInvariant();
						bool flag12 = text7.Contains("steam");
						if (flag12)
						{
							num += 10f;
						}
						else
						{
							bool flag13 = text7.Contains("rift") || text7.Contains("pcvr");
							if (flag13)
							{
								num += 5f;
							}
						}
					}
				}
				catch
				{
				}
				try
				{
					bool flag14 = speedRig != null;
					if (flag14)
					{
						float rigSpeed = this.GetRigSpeed(speedRig);
						bool flag15 = rigSpeed > 25f;
						if (flag15)
						{
							num += 10f;
						}
					}
				}
				catch
				{
				}
				result = Mathf.Clamp(num, 0f, 100f);
			}
			return result;
		}

		// Token: 0x0600002F RID: 47 RVA: 0x00003F48 File Offset: 0x00002148
		private float GetRigSpeed(VRRig rig)
		{
			bool flag = rig == null || this.speedMphText == null;
			float result;
			if (flag)
			{
				result = 0f;
			}
			else
			{
				string text = this.speedMphText.text;
				bool flag2 = text.Contains(":");
				if (flag2)
				{
					string[] array = text.Split(new char[]
					{
						':'
					});
					bool flag3 = array.Length > 1;
					if (flag3)
					{
						string s = array[1].Trim();
						float result2;
						bool flag4 = float.TryParse(s, out result2);
						if (flag4)
						{
							return result2;
						}
					}
				}
				result = 0f;
			}
			return result;
		}

		// Token: 0x06000030 RID: 48 RVA: 0x00003FE4 File Offset: 0x000021E4
		private void UpdateCheaterDetection(VRRig rig, VRRig speedRig)
		{
			bool flag = this.cheaterPanel == null || this.cheaterTitleText == null || this.cheaterPercentText == null || this.cheaterBarFillImage == null;
			if (!flag)
			{
				bool flag2 = this.uiCanvasGroup == null || !this.uiCanvasGroup.interactable;
				if (flag2)
				{
					this.cheaterPanel.SetActive(false);
				}
				else
				{
					bool flag3 = !this.cheaterPanel.activeSelf;
					if (flag3)
					{
						this.cheaterPanel.SetActive(true);
					}
					bool flag4 = rig == null;
					if (flag4)
					{
						this.cheaterPercentText.text = "---";
						this.cheaterPercentText.color = this.textLight;
						this.cheaterTitleText.text = "Select Player";
						this.cheaterTitleText.color = new Color(0.7f, 0.7f, 0.7f, 1f);
						RectTransform component = this.cheaterBarFillImage.GetComponent<RectTransform>();
						component.sizeDelta = new Vector2(0f, component.sizeDelta.y);
						this.cheaterBarFillImage.color = Color.gray;
						this.currentCheaterPercent = 0f;
						this.displayedCheaterPercent = 0f;
						this.lastCheckedRig = null;
					}
					else
					{
						this.cheaterTitleText.text = "Cheater?";
						this.cheaterTitleText.color = new Color(1f, 0.35f, 0.35f, 1f);
						bool flag5 = rig != this.lastCheckedRig;
						if (flag5)
						{
							this.lastCheckedRig = rig;
							this.currentCheaterPercent = this.CalculateCheaterPercent(rig, speedRig);
						}
						else
						{
							this.currentCheaterPercent = this.CalculateCheaterPercent(rig, speedRig);
						}
						this.displayedCheaterPercent = Mathf.Lerp(this.displayedCheaterPercent, this.currentCheaterPercent, Time.deltaTime * 5f);
						this.cheaterPercentText.text = string.Format("{0}%", Mathf.RoundToInt(this.displayedCheaterPercent));
						float num = this.displayedCheaterPercent / 100f;
						RectTransform component2 = this.cheaterBarFillImage.GetComponent<RectTransform>();
						float b = num * (this.cheaterBarWidth - 8f);
						Vector2 sizeDelta = component2.sizeDelta;
						component2.sizeDelta = new Vector2(Mathf.Lerp(sizeDelta.x, b, Time.deltaTime * 5f), sizeDelta.y);
						bool flag6 = this.displayedCheaterPercent < 25f;
						Color color;
						if (flag6)
						{
							color = Color.green;
						}
						else
						{
							bool flag7 = this.displayedCheaterPercent < 50f;
							if (flag7)
							{
								color = Color.yellow;
							}
							else
							{
								bool flag8 = this.displayedCheaterPercent < 75f;
								if (flag8)
								{
									color = new Color(1f, 0.5f, 0f);
								}
								else
								{
									color = Color.red;
								}
							}
						}
						this.cheaterBarFillImage.color = color;
						this.cheaterPercentText.color = color;
					}
				}
			}
		}

		// Token: 0x06000031 RID: 49 RVA: 0x000042F0 File Offset: 0x000024F0
		private void ResetCheaterDisplay()
		{
			bool flag = this.cheaterPanel != null;
			if (flag)
			{
				this.cheaterPanel.SetActive(false);
			}
			bool flag2 = this.cheaterPercentText != null;
			if (flag2)
			{
				this.cheaterPercentText.text = "0%";
			}
			bool flag3 = this.cheaterBarFillImage != null;
			if (flag3)
			{
				RectTransform component = this.cheaterBarFillImage.GetComponent<RectTransform>();
				component.sizeDelta = new Vector2(0f, component.sizeDelta.y);
				this.cheaterBarFillImage.color = Color.green;
			}
			bool flag4 = this.cheaterPercentText != null;
			if (flag4)
			{
				this.cheaterPercentText.color = Color.green;
			}
			this.currentCheaterPercent = 0f;
			this.displayedCheaterPercent = 0f;
			this.lastCheckedRig = null;
		}

		// Token: 0x06000032 RID: 50 RVA: 0x000043C8 File Offset: 0x000025C8
		private void ShowPanel(bool show)
		{
			bool flag = this.uiCanvasGroup == null;
			if (!flag)
			{
				this.uiCanvasGroup.interactable = show;
				this.uiCanvasGroup.blocksRaycasts = show;
			}
		}

		// Token: 0x06000033 RID: 51 RVA: 0x00004404 File Offset: 0x00002604
		private void SpawnUIPanel()
		{
			bool flag = this.uiPanel != null;
			if (flag)
			{
				this.ShowPanel(true);
			}
			else
			{
				this.currentPage = 1;
				this.uiPanel = new GameObject("VaelCatcher");
				Canvas canvas = this.uiPanel.AddComponent<Canvas>();
				canvas.renderMode = RenderMode.WorldSpace;
				canvas.worldCamera = Camera.main;
				this.uiCanvasGroup = this.uiPanel.AddComponent<CanvasGroup>();
				this.uiCanvasGroup.interactable = true;
				this.uiCanvasGroup.blocksRaycasts = true;
				CanvasScaler canvasScaler = this.uiPanel.AddComponent<CanvasScaler>();
				canvasScaler.dynamicPixelsPerUnit = 1000f;
				this.uiPanel.AddComponent<GraphicRaycaster>();
				RectTransform component = this.uiPanel.GetComponent<RectTransform>();
				component.sizeDelta = new Vector2(320f, 300f);
				this.uiPanel.transform.localScale = Vector3.one * 0.0015f;
				this.CreateSidebar();
				GameObject gameObject = new GameObject("Background");
				gameObject.transform.SetParent(this.uiPanel.transform, false);
				this.mainBgImage = gameObject.AddComponent<Image>();
				this.mainBgImage.color = new Color(this.panelColor.r, this.panelColor.g, this.panelColor.b, this.menuOpacity);
				this.mainBgImage.sprite = this.CreateOptimizedRoundedSprite(512, 512, 12);
				this.mainBgImage.type = Image.Type.Sliced;
				RectTransform component2 = gameObject.GetComponent<RectTransform>();
				component2.sizeDelta = new Vector2(210f, 280f);
				component2.anchoredPosition = new Vector2(25f, 0f);
				Outline outline = gameObject.AddComponent<Outline>();
				outline.effectColor = new Color(this.separatorColor.r, this.separatorColor.g, this.separatorColor.b, 0.5f);
				outline.effectDistance = new Vector2(2f, 2f);
				Shadow shadow = gameObject.AddComponent<Shadow>();
				shadow.effectColor = new Color(0f, 0f, 0f, 0.25f);
				shadow.effectDistance = new Vector2(3f, -3f);
				this.disconnectBtn = new GameObject("DisconnectButton");
				this.disconnectBtn.transform.SetParent(this.uiPanel.transform, false);
				Image image = this.disconnectBtn.AddComponent<Image>();
				image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 8);
				image.type = Image.Type.Sliced;
				image.color = new Color(0.7f, 0.2f, 0.3f, 0.95f);
				image.raycastTarget = true;
				RectTransform component3 = this.disconnectBtn.GetComponent<RectTransform>();
				component3.sizeDelta = new Vector2(140f, 38f);
				component3.anchoredPosition = new Vector2(25f, 165f);
				component3.localScale = Vector3.one;
				GameObject gameObject2 = new GameObject("Text");
				gameObject2.transform.SetParent(this.disconnectBtn.transform, false);
				TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
				textMeshProUGUI.text = "Disconnect";
				textMeshProUGUI.fontSize = 12f;
				textMeshProUGUI.alignment = TextAlignmentOptions.Center;
				textMeshProUGUI.color = Color.white;
				RectTransform component4 = gameObject2.GetComponent<RectTransform>();
				component4.anchorMin = Vector2.zero;
				component4.anchorMax = Vector2.one;
				component4.offsetMin = Vector2.zero;
				component4.offsetMax = Vector2.zero;
				Button button = this.disconnectBtn.AddComponent<Button>();
				button.transition = Selectable.Transition.ColorTint;
				button.onClick.AddListener(delegate()
				{
					this.PlayClickFeedback();
					this.OpenConfirmPopup("Leave the lobby?", new Action(this.ConfirmDisconnect));
				});
				this.CreateConfirmPopup();
				this.CreateCheaterPanel(gameObject.transform);
				this.CreateInfoText(gameObject.transform, ref this.titleText, "Vael Catcher", new Vector2(0f, 115f), 14f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.pageText, string.Format("Page {0}/{1}", this.currentPage, this.maxPage), new Vector2(0f, 100f), 9f, FontStyles.Italic);
				this.pageText.color = new Color(0.6f, 0.6f, 0.6f, 1f);
				this.CreateInfoText(gameObject.transform, ref this.pageNameText, this.pageNames[0], new Vector2(0f, 85f), 11f, FontStyles.Bold);
				this.pageNameText.color = this.activeGreen;
				this.CreateInfoText(gameObject.transform, ref this.playerNameText, "Name: Unknown", new Vector2(0f, 55f), 12f, FontStyles.Italic);
				this.playerColorCodeText = this.CreateColorText(gameObject.transform, "Color: ---", new Vector2(0f, 40f), 11f);
				this.CreateInfoText(gameObject.transform, ref this.playerPlatformText, "Platform: Loading...", new Vector2(0f, 25f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.playerFPSText, "FPS: Select Player", new Vector2(0f, 5f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.playerPingText, "Ping: --- ms", new Vector2(0f, -14f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.playerCreationDateText, "Creation: Loading...", new Vector2(0f, -36f), 9f, FontStyles.Normal);
				this.CreateInfoText(gameObject.transform, ref this.playerModsText, "Mods/Cheats: None", new Vector2(0f, 40f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.playerCosmeticsText, "Cosmetics: None", new Vector2(0f, 10f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.importantChecksTitleText, "Speed Detection", new Vector2(0f, 60f), 12f, FontStyles.Bold);
				this.CreateInfoText(gameObject.transform, ref this.speedMphText, "Speed mph: ---", new Vector2(0f, 30f), 11f, FontStyles.Normal);
				this.CreateInfoText(gameObject.transform, ref this.lobbyCodeText, "Lobby: ---", new Vector2(0f, 65f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.lobbyPlayerCountText, "Players: ---", new Vector2(0f, 45f), 10f, FontStyles.Normal);
				this.CreateInfoText(gameObject.transform, ref this.lobbyGameModeText, "Game Mode: ---", new Vector2(0f, 25f), 10f, FontStyles.Normal);
				this.CreateInfoText(gameObject.transform, ref this.lobbyQueueText, "Queue: ---", new Vector2(0f, 5f), 10f, FontStyles.Normal);
				this.CreateInfoText(gameObject.transform, ref this.lobbyMasterText, "Master: ---", new Vector2(0f, -15f), 10f, FontStyles.Normal);
				this.gunRayToggle = this.CreateSettingToggle(gameObject.transform, "Gun Lib", new Vector2(0f, 60f), new Action(this.ToggleGunRay));
				this.autoLockToggle = this.CreateSettingToggle(gameObject.transform, "Auto Lock", new Vector2(0f, 35f), new Action(this.ToggleAutoLock));
				this.opacityLowBtn = this.CreateSettingButton(gameObject.transform, "Low", new Vector2(-45f, 5f), delegate
				{
					this.SetOpacity(0.7f);
				});
				this.opacityMedBtn = this.CreateSettingButton(gameObject.transform, "Med", new Vector2(0f, 5f), delegate
				{
					this.SetOpacity(0.85f);
				});
				this.opacityHighBtn = this.CreateSettingButton(gameObject.transform, "High", new Vector2(45f, 5f), delegate
				{
					this.SetOpacity(0.95f);
				});
				this.resetBtn = this.CreateSettingButton(gameObject.transform, "Reset All", new Vector2(0f, -25f), new Action(this.ResetSettings));
				this.reportBtn = this.CreateActionButton(gameObject.transform, "Report Player", new Vector2(0f, 50f), delegate
				{
					this.OpenReportConfirm();
				}, new Color(0.8f, 0.3f, 0.2f, 1f));
				this.muteBtn = this.CreateActionButton(gameObject.transform, "Mute Player", new Vector2(0f, 15f), delegate
				{
					this.OpenMuteConfirm();
				}, new Color(0.6f, 0.5f, 0.2f, 1f));
				this.blockBtn = this.CreateActionButton(gameObject.transform, "Block Player", new Vector2(0f, -20f), delegate
				{
					this.OpenBlockConfirm();
				}, new Color(0.5f, 0.2f, 0.2f, 1f));
				this.CreateInfoText(gameObject.transform, ref this.reportStatusText, "Select a player first", new Vector2(0f, -55f), 9f, FontStyles.Italic);
				this.reportStatusText.color = new Color(0.7f, 0.7f, 0.7f, 1f);
				this.CreateThemesPage(gameObject.transform);
				this.CreateInfoText(gameObject.transform, ref this.creditsText, "Vael Private Checker V1.0.4", new Vector2(0f, -70f), 5f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.creditsOwnerText, "Owner: Duv14", new Vector2(0f, 55f), 11f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.creditsCoOwnerText, "Co-Owner: Lumo and Zaxy", new Vector2(0f, 35f), 10f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.creditsHeadDevText, "Head dev: Duv14", new Vector2(0f, 15f), 10f, FontStyles.Italic);
				this.CreateInfoText(gameObject.transform, ref this.creditsThankYouText, "Thank you for using our menu!", new Vector2(0f, -15f), 9f, FontStyles.Italic);
				this.separatorLine = new GameObject("Separator");
				this.separatorLine.transform.SetParent(gameObject.transform, false);
				Image image2 = this.separatorLine.AddComponent<Image>();
				image2.color = this.separatorColor;
				RectTransform component5 = this.separatorLine.GetComponent<RectTransform>();
				component5.sizeDelta = new Vector2(180f, 1f);
				component5.anchoredPosition = new Vector2(0f, -85f);
				this.UpdateInfoVisibility();
				this.RefreshThemeUI();
				this.fadeRoutine = base.StartCoroutine(this.FadeAllGraphics(this.uiPanel, 0f, 1f, 0.12f, false));
			}
		}

		// Token: 0x06000034 RID: 52 RVA: 0x00004F8C File Offset: 0x0000318C
		private void CreateThemesPage(Transform parent)
		{
			this.CreateInfoText(parent, ref this.themeTitleText, "Select Theme", new Vector2(0f, 65f), 12f, FontStyles.Bold);
			this.themeButtons.Clear();
			float num = 35f;
			float num2 = 28f;
			for (int i = 0; i < this.themeNames.Length; i++)
			{
				int themeIdx = i;
				GameObject gameObject = new GameObject("ThemeBtn_" + this.themeNames[i]);
				gameObject.transform.SetParent(parent, false);
				Image image = gameObject.AddComponent<Image>();
				image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
				image.type = Image.Type.Sliced;
				image.color = ((i == this.currentTheme) ? this.activeGreen : this.elementDark);
				RectTransform component = gameObject.GetComponent<RectTransform>();
				component.sizeDelta = new Vector2(140f, 24f);
				component.anchoredPosition = new Vector2(0f, num - (float)i * num2);
				GameObject gameObject2 = new GameObject("Text");
				gameObject2.transform.SetParent(gameObject.transform, false);
				TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
				textMeshProUGUI.text = this.themeNames[i];
				textMeshProUGUI.fontSize = 10f;
				textMeshProUGUI.alignment = TextAlignmentOptions.Center;
				textMeshProUGUI.color = this.textLight;
				RectTransform component2 = gameObject2.GetComponent<RectTransform>();
				component2.anchorMin = Vector2.zero;
				component2.anchorMax = Vector2.one;
				component2.offsetMin = Vector2.zero;
				component2.offsetMax = Vector2.zero;
				Button button = gameObject.AddComponent<Button>();
				button.onClick.AddListener(delegate()
				{
					this.PlayClickFeedback();
					this.SetTheme(themeIdx);
				});
				this.themeButtons.Add(gameObject);
			}
		}

		// Token: 0x06000035 RID: 53 RVA: 0x00005188 File Offset: 0x00003388
		private void CreateSidebar()
		{
			this.sidebarPanel = new GameObject("Sidebar");
			this.sidebarPanel.transform.SetParent(this.uiPanel.transform, false);
			this.sidebarBgImage = this.sidebarPanel.AddComponent<Image>();
			this.sidebarBgImage.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			this.sidebarBgImage.type = Image.Type.Sliced;
			this.sidebarBgImage.color = this.sidebarColor;
			RectTransform component = this.sidebarPanel.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(32f, 250f);
			component.anchoredPosition = new Vector2(-125f, 0f);
			Outline outline = this.sidebarPanel.AddComponent<Outline>();
			outline.effectColor = new Color(this.separatorColor.r, this.separatorColor.g, this.separatorColor.b, 0.4f);
			outline.effectDistance = new Vector2(1.5f, 1.5f);
			this.sidebarButtons.Clear();
			this.sidebarButtonImages.Clear();
			this.sidebarIconImages.Clear();
			float num = 100f;
			float num2 = 26f;
			for (int i = 0; i < this.maxPage; i++)
			{
				int pageIndex = i;
				GameObject gameObject = new GameObject(string.Format("PageBtn_{0}", i + 1));
				gameObject.transform.SetParent(this.sidebarPanel.transform, false);
				Image image = gameObject.AddComponent<Image>();
				image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 4);
				image.type = Image.Type.Sliced;
				image.color = ((i == 0) ? this.sidebarBtnActiveColor : this.sidebarBtnColor);
				RectTransform component2 = gameObject.GetComponent<RectTransform>();
				component2.sizeDelta = new Vector2(26f, 22f);
				component2.anchoredPosition = new Vector2(0f, num - (float)i * num2);
				GameObject gameObject2 = new GameObject("Icon");
				gameObject2.transform.SetParent(gameObject.transform, false);
				Image image2 = gameObject2.AddComponent<Image>();
				image2.color = this.textLight;
				image2.preserveAspect = true;
				RectTransform component3 = gameObject2.GetComponent<RectTransform>();
				component3.anchorMin = new Vector2(0.5f, 0.5f);
				component3.anchorMax = new Vector2(0.5f, 0.5f);
				component3.pivot = new Vector2(0.5f, 0.5f);
				component3.sizeDelta = new Vector2(14f, 14f);
				component3.anchoredPosition = Vector2.zero;
				this.sidebarIconImages.Add(image2);
				Button button = gameObject.AddComponent<Button>();
				button.onClick.AddListener(delegate()
				{
					this.PlayClickFeedback();
					this.GoToPage(pageIndex + 1);
				});
				this.sidebarButtons.Add(gameObject);
				this.sidebarButtonImages.Add(image);
			}
			this.UpdateSidebarIcons();
		}

		// Token: 0x06000036 RID: 54 RVA: 0x000054C0 File Offset: 0x000036C0
		private void CreateCheaterPanel(Transform parent)
		{
			this.cheaterPanel = new GameObject("CheaterPanel");
			this.cheaterPanel.transform.SetParent(parent, false);
			this.cheaterPanelImage = this.cheaterPanel.AddComponent<Image>();
			this.cheaterPanelImage.sprite = this.CreateOptimizedRoundedSprite(512, 512, 8);
			this.cheaterPanelImage.type = Image.Type.Sliced;
			this.cheaterPanelImage.color = this.cheaterPanelColor;
			RectTransform component = this.cheaterPanel.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(this.cheaterBarWidth + 20f, 32f);
			component.anchoredPosition = new Vector2(0f, -108f);
			this.cheaterPanelOutlineComp = this.cheaterPanel.AddComponent<Outline>();
			this.cheaterPanelOutlineComp.effectColor = this.cheaterPanelOutline;
			this.cheaterPanelOutlineComp.effectDistance = new Vector2(1.5f, 1.5f);
			GameObject gameObject = new GameObject("CheaterTitle");
			gameObject.transform.SetParent(this.cheaterPanel.transform, false);
			this.cheaterTitleText = gameObject.AddComponent<TextMeshProUGUI>();
			this.cheaterTitleText.text = "Cheater?";
			this.cheaterTitleText.fontSize = 8f;
			this.cheaterTitleText.fontStyle = FontStyles.Bold;
			this.cheaterTitleText.alignment = TextAlignmentOptions.Left;
			this.cheaterTitleText.color = new Color(1f, 0.35f, 0.35f, 1f);
			RectTransform component2 = gameObject.GetComponent<RectTransform>();
			component2.sizeDelta = new Vector2(70f, 20f);
			component2.anchoredPosition = new Vector2(-30f, 8f);
			GameObject gameObject2 = new GameObject("CheaterPercent");
			gameObject2.transform.SetParent(this.cheaterPanel.transform, false);
			this.cheaterPercentText = gameObject2.AddComponent<TextMeshProUGUI>();
			this.cheaterPercentText.text = "0%";
			this.cheaterPercentText.fontSize = 9f;
			this.cheaterPercentText.fontStyle = FontStyles.Bold;
			this.cheaterPercentText.alignment = TextAlignmentOptions.Right;
			this.cheaterPercentText.color = Color.green;
			RectTransform component3 = gameObject2.GetComponent<RectTransform>();
			component3.sizeDelta = new Vector2(50f, 20f);
			component3.anchoredPosition = new Vector2(45f, 8f);
			this.cheaterBarBg = new GameObject("CheaterBarBg");
			this.cheaterBarBg.transform.SetParent(this.cheaterPanel.transform, false);
			Image image = this.cheaterBarBg.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 4);
			image.type = Image.Type.Sliced;
			image.color = new Color(this.elementDark.r * 0.5f, this.elementDark.g * 0.5f, this.elementDark.b * 0.5f, 1f);
			RectTransform component4 = this.cheaterBarBg.GetComponent<RectTransform>();
			component4.sizeDelta = new Vector2(this.cheaterBarWidth, 8f);
			component4.anchoredPosition = new Vector2(0f, -8f);
			this.cheaterBarFill = new GameObject("CheaterBarFill");
			this.cheaterBarFill.transform.SetParent(this.cheaterBarBg.transform, false);
			this.cheaterBarFillImage = this.cheaterBarFill.AddComponent<Image>();
			this.cheaterBarFillImage.sprite = this.CreateOptimizedRoundedSprite(512, 512, 4);
			this.cheaterBarFillImage.type = Image.Type.Sliced;
			this.cheaterBarFillImage.color = Color.green;
			RectTransform component5 = this.cheaterBarFill.GetComponent<RectTransform>();
			component5.anchorMin = new Vector2(0f, 0f);
			component5.anchorMax = new Vector2(0f, 1f);
			component5.pivot = new Vector2(0f, 0.5f);
			component5.anchoredPosition = new Vector2(4f, 0f);
			component5.sizeDelta = new Vector2(0f, 5f);
			this.cheaterPanel.SetActive(true);
		}

		// Token: 0x06000037 RID: 55 RVA: 0x0000591C File Offset: 0x00003B1C
		private void GoToPage(int page)
		{
			this.currentPage = Mathf.Clamp(page, 1, this.maxPage);
			this.pageText.text = string.Format("Page {0}/{1}", this.currentPage, this.maxPage);
			bool flag = this.pageNameText != null && this.currentPage >= 1 && this.currentPage <= this.pageNames.Length;
			if (flag)
			{
				this.pageNameText.text = this.pageNames[this.currentPage - 1];
			}
			for (int i = 0; i < this.sidebarButtonImages.Count; i++)
			{
				bool flag2 = this.sidebarButtonImages[i] != null;
				if (flag2)
				{
					this.sidebarButtonImages[i].color = ((i == this.currentPage - 1) ? this.sidebarBtnActiveColor : this.sidebarBtnColor);
				}
			}
			this.UpdateInfoVisibility();
		}

		// Token: 0x06000038 RID: 56 RVA: 0x00005A1C File Offset: 0x00003C1C
		private void CreateConfirmPopup()
		{
			this.confirmPopup = new GameObject("ConfirmPopup");
			this.confirmPopup.transform.SetParent(this.uiPanel.transform, false);
			Image image = this.confirmPopup.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 10);
			image.type = Image.Type.Sliced;
			image.color = new Color(this.panelColor.r * 0.9f, this.panelColor.g * 0.9f, this.panelColor.b * 0.9f, 0.98f);
			RectTransform component = this.confirmPopup.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(180f, 100f);
			component.anchoredPosition = new Vector2(25f, 0f);
			Outline outline = this.confirmPopup.AddComponent<Outline>();
			outline.effectColor = this.separatorColor;
			outline.effectDistance = new Vector2(2f, 2f);
			GameObject gameObject = new GameObject("Message");
			gameObject.transform.SetParent(this.confirmPopup.transform, false);
			this.confirmMessageText = gameObject.AddComponent<TextMeshProUGUI>();
			this.confirmMessageText.text = "Are you sure?";
			this.confirmMessageText.fontSize = 11f;
			this.confirmMessageText.alignment = TextAlignmentOptions.Center;
			this.confirmMessageText.color = this.textLight;
			RectTransform component2 = gameObject.GetComponent<RectTransform>();
			component2.anchorMin = new Vector2(0f, 0.5f);
			component2.anchorMax = new Vector2(1f, 1f);
			component2.offsetMin = new Vector2(10f, 0f);
			component2.offsetMax = new Vector2(-10f, -10f);
			GameObject gameObject2 = new GameObject("YesBtn");
			gameObject2.transform.SetParent(this.confirmPopup.transform, false);
			Image image2 = gameObject2.AddComponent<Image>();
			image2.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			image2.type = Image.Type.Sliced;
			image2.color = new Color(0.15f, 0.6f, 0.2f, 1f);
			RectTransform component3 = gameObject2.GetComponent<RectTransform>();
			component3.sizeDelta = new Vector2(70f, 28f);
			component3.anchoredPosition = new Vector2(-45f, -30f);
			Button button = gameObject2.AddComponent<Button>();
			button.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				this.ExecutePendingAction();
			});
			this.confirmYesBtn = gameObject2;
			TextMeshProUGUI textMeshProUGUI = new GameObject("Text").AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.transform.SetParent(gameObject2.transform, false);
			textMeshProUGUI.text = "Yes";
			textMeshProUGUI.fontSize = 10f;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = Color.white;
			textMeshProUGUI.rectTransform.anchorMin = Vector2.zero;
			textMeshProUGUI.rectTransform.anchorMax = Vector2.one;
			textMeshProUGUI.rectTransform.offsetMin = (textMeshProUGUI.rectTransform.offsetMax = Vector2.zero);
			GameObject gameObject3 = new GameObject("NoBtn");
			gameObject3.transform.SetParent(this.confirmPopup.transform, false);
			Image image3 = gameObject3.AddComponent<Image>();
			image3.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			image3.type = Image.Type.Sliced;
			image3.color = new Color(0.6f, 0.15f, 0.15f, 1f);
			RectTransform component4 = gameObject3.GetComponent<RectTransform>();
			component4.sizeDelta = new Vector2(70f, 28f);
			component4.anchoredPosition = new Vector2(45f, -30f);
			Button button2 = gameObject3.AddComponent<Button>();
			button2.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				this.CloseConfirmPopup();
			});
			this.confirmNoBtn = gameObject3;
			TextMeshProUGUI textMeshProUGUI2 = new GameObject("Text").AddComponent<TextMeshProUGUI>();
			textMeshProUGUI2.transform.SetParent(gameObject3.transform, false);
			textMeshProUGUI2.text = "No";
			textMeshProUGUI2.fontSize = 10f;
			textMeshProUGUI2.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI2.color = Color.white;
			textMeshProUGUI2.rectTransform.anchorMin = Vector2.zero;
			textMeshProUGUI2.rectTransform.anchorMax = Vector2.one;
			textMeshProUGUI2.rectTransform.offsetMin = (textMeshProUGUI2.rectTransform.offsetMax = Vector2.zero);
			this.confirmPopup.SetActive(false);
		}

		// Token: 0x06000039 RID: 57 RVA: 0x00005EFC File Offset: 0x000040FC
		private void OpenConfirmPopup(string message, Action onConfirm)
		{
			bool flag = this.popupOpen;
			if (!flag)
			{
				this.popupOpen = true;
				this.pendingConfirmAction = onConfirm;
				bool flag2 = this.confirmMessageText != null;
				if (flag2)
				{
					this.confirmMessageText.text = message;
				}
				this.confirmPopup.SetActive(true);
				this.confirmPopup.transform.localScale = Vector3.zero;
				base.StartCoroutine(this.ScalePopup(this.confirmPopup, Vector3.one, 0.12f, null));
			}
		}

		// Token: 0x0600003A RID: 58 RVA: 0x00005F84 File Offset: 0x00004184
		private void CloseConfirmPopup()
		{
			bool flag = !this.popupOpen;
			if (!flag)
			{
				this.popupOpen = false;
				this.pendingConfirmAction = null;
				base.StartCoroutine(this.ScalePopup(this.confirmPopup, Vector3.zero, 0.12f, delegate
				{
					this.confirmPopup.SetActive(false);
				}));
			}
		}

		// Token: 0x0600003B RID: 59 RVA: 0x00005FD8 File Offset: 0x000041D8
		private void ExecutePendingAction()
		{
			Action action = this.pendingConfirmAction;
			this.CloseConfirmPopup();
			if (action != null)
			{
				action();
			}
		}

		// Token: 0x0600003C RID: 60 RVA: 0x00006000 File Offset: 0x00004200
		private void OpenReportConfirm()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				this.OpenConfirmPopup("Report " + str + "?", new Action(this.ReportPlayer));
			}
		}

		// Token: 0x0600003D RID: 61 RVA: 0x00006088 File Offset: 0x00004288
		private void OpenMuteConfirm()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				this.OpenConfirmPopup("Mute " + str + "?", new Action(this.MutePlayer));
			}
		}

		// Token: 0x0600003E RID: 62 RVA: 0x00006110 File Offset: 0x00004310
		private void OpenBlockConfirm()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				this.OpenConfirmPopup("Block " + str + "?", new Action(this.BlockPlayer));
			}
		}

		// Token: 0x0600003F RID: 63 RVA: 0x00006198 File Offset: 0x00004398
		private GameObject CreateColorText(Transform parent, string content, Vector2 pos, float fontSize)
		{
			GameObject gameObject = new GameObject("ColorText");
			gameObject.transform.SetParent(parent, false);
			TextMeshProUGUI textMeshProUGUI = gameObject.AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.text = content;
			textMeshProUGUI.fontSize = fontSize;
			textMeshProUGUI.fontStyle = FontStyles.Normal;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = this.textLight;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(200f, 30f);
			component.anchoredPosition = pos;
			return gameObject;
		}

		// Token: 0x06000040 RID: 64 RVA: 0x00006224 File Offset: 0x00004424
		private void CreateInfoText(Transform parent, ref TextMeshProUGUI textRef, string content, Vector2 pos, float fontSize, FontStyles fontStyle)
		{
			GameObject gameObject = new GameObject("InfoText");
			gameObject.transform.SetParent(parent, false);
			textRef = gameObject.AddComponent<TextMeshProUGUI>();
			textRef.text = content;
			textRef.fontSize = fontSize;
			textRef.fontStyle = fontStyle;
			textRef.alignment = TextAlignmentOptions.Center;
			textRef.color = this.textLight;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(200f, 30f);
			component.anchoredPosition = pos;
		}

		// Token: 0x06000041 RID: 65 RVA: 0x000062B4 File Offset: 0x000044B4
		private GameObject CreateSettingToggle(Transform parent, string label, Vector2 pos, Action onClick)
		{
			GameObject gameObject = new GameObject(label + "Toggle");
			gameObject.transform.SetParent(parent, false);
			Image image = gameObject.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			image.type = Image.Type.Sliced;
			image.color = this.elementDark;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(130f, 22f);
			component.anchoredPosition = pos;
			GameObject gameObject2 = new GameObject("Text");
			gameObject2.transform.SetParent(gameObject.transform, false);
			TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.text = label;
			textMeshProUGUI.fontSize = 8f;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = this.textLight;
			RectTransform component2 = gameObject2.GetComponent<RectTransform>();
			component2.anchorMin = Vector2.zero;
			component2.anchorMax = Vector2.one;
			component2.offsetMin = Vector2.zero;
			component2.offsetMax = Vector2.zero;
			Button button = gameObject.AddComponent<Button>();
			button.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				Action onClick2 = onClick;
				if (onClick2 != null)
				{
					onClick2();
				}
			});
			return gameObject;
		}

		// Token: 0x06000042 RID: 66 RVA: 0x00006410 File Offset: 0x00004610
		private GameObject CreateSettingButton(Transform parent, string label, Vector2 pos, Action onClick)
		{
			GameObject gameObject = new GameObject(label + "Btn");
			gameObject.transform.SetParent(parent, false);
			Image image = gameObject.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 5);
			image.type = Image.Type.Sliced;
			image.color = this.elementDark;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(42f, 18f);
			component.anchoredPosition = pos;
			GameObject gameObject2 = new GameObject("Text");
			gameObject2.transform.SetParent(gameObject.transform, false);
			TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.text = label;
			textMeshProUGUI.fontSize = 7f;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = this.textLight;
			RectTransform component2 = gameObject2.GetComponent<RectTransform>();
			component2.anchorMin = Vector2.zero;
			component2.anchorMax = Vector2.one;
			component2.offsetMin = Vector2.zero;
			component2.offsetMax = Vector2.zero;
			Button button = gameObject.AddComponent<Button>();
			button.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				Action onClick2 = onClick;
				if (onClick2 != null)
				{
					onClick2();
				}
			});
			return gameObject;
		}

		// Token: 0x06000043 RID: 67 RVA: 0x0000656C File Offset: 0x0000476C
		private GameObject CreateActionButton(Transform parent, string label, Vector2 pos, Action onClick, Color btnColor)
		{
			GameObject gameObject = new GameObject(label + "Btn");
			gameObject.transform.SetParent(parent, false);
			Image image = gameObject.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			image.type = Image.Type.Sliced;
			image.color = btnColor;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(130f, 26f);
			component.anchoredPosition = pos;
			GameObject gameObject2 = new GameObject("Text");
			gameObject2.transform.SetParent(gameObject.transform, false);
			TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.text = label;
			textMeshProUGUI.fontSize = 9f;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = Color.white;
			RectTransform component2 = gameObject2.GetComponent<RectTransform>();
			component2.anchorMin = Vector2.zero;
			component2.anchorMax = Vector2.one;
			component2.offsetMin = Vector2.zero;
			component2.offsetMax = Vector2.zero;
			Button button = gameObject.AddComponent<Button>();
			button.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				Action onClick2 = onClick;
				if (onClick2 != null)
				{
					onClick2();
				}
			});
			return gameObject;
		}

		// Token: 0x06000044 RID: 68 RVA: 0x000066C4 File Offset: 0x000048C4
		private void ReportPlayer()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				bool flag3 = this.reportStatusText != null;
				if (flag3)
				{
					this.reportStatusText.text = "Reported: " + str;
				}
			}
		}

		// Token: 0x06000045 RID: 69 RVA: 0x00006750 File Offset: 0x00004950
		private void MutePlayer()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				bool flag3 = this.reportStatusText != null;
				if (flag3)
				{
					this.reportStatusText.text = "Muted: " + str;
				}
			}
		}

		// Token: 0x06000046 RID: 70 RVA: 0x000067DC File Offset: 0x000049DC
		private void BlockPlayer()
		{
			bool flag = this.selectedRig == null;
			if (flag)
			{
				bool flag2 = this.reportStatusText != null;
				if (flag2)
				{
					this.reportStatusText.text = "No player selected!";
				}
			}
			else
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				bool flag3 = this.reportStatusText != null;
				if (flag3)
				{
					this.reportStatusText.text = "Blocked: " + str;
				}
			}
		}

		// Token: 0x06000047 RID: 71 RVA: 0x00006866 File Offset: 0x00004A66
		public void OnJoinedRoom(Room newRoom)
		{
			this.lastRoom = newRoom.Name;
		}

		// Token: 0x06000048 RID: 72 RVA: 0x00006878 File Offset: 0x00004A78
		private void ReconnectToLastRoom()
		{
			bool flag = string.IsNullOrEmpty(this.lastRoom);
			if (!flag)
			{
				PhotonNetwork.JoinRoom(this.lastRoom, null);
			}
		}

		// Token: 0x06000049 RID: 73 RVA: 0x000068A4 File Offset: 0x00004AA4
		private void ToggleGunRay()
		{
			this.gunRayEnabled = !this.gunRayEnabled;
			bool flag = this.gunRayToggle != null;
			if (flag)
			{
				this.gunRayToggle.GetComponent<Image>().color = (this.gunRayEnabled ? this.activeGreen : this.elementDark);
			}
			base.StartCoroutine(this.PlayRipple(this.gunRayToggle.GetComponent<Image>()));
		}

		// Token: 0x0600004A RID: 74 RVA: 0x00006914 File Offset: 0x00004B14
		private void ToggleAutoLock()
		{
			this.autoLockEnabled = !this.autoLockEnabled;
			bool flag = this.autoLockToggle != null;
			if (flag)
			{
				this.autoLockToggle.GetComponent<Image>().color = (this.autoLockEnabled ? this.activeGreen : this.elementDark);
			}
			base.StartCoroutine(this.PlayRipple(this.autoLockToggle.GetComponent<Image>()));
		}

		// Token: 0x0600004B RID: 75 RVA: 0x00006984 File Offset: 0x00004B84
		private void SetOpacity(float opacity)
		{
			this.menuOpacity = opacity;
			bool flag = this.mainBgImage != null;
			if (flag)
			{
				Color color = this.mainBgImage.color;
				this.mainBgImage.color = new Color(color.r, color.g, color.b, opacity);
			}
			bool flag2 = this.opacityLowBtn != null;
			if (flag2)
			{
				this.opacityLowBtn.GetComponent<Image>().color = ((opacity == 0.7f) ? this.activeGreen : this.elementDark);
			}
			bool flag3 = this.opacityMedBtn != null;
			if (flag3)
			{
				this.opacityMedBtn.GetComponent<Image>().color = ((opacity == 0.85f) ? this.activeGreen : this.elementDark);
			}
			bool flag4 = this.opacityHighBtn != null;
			if (flag4)
			{
				this.opacityHighBtn.GetComponent<Image>().color = ((opacity == 0.95f) ? this.activeGreen : this.elementDark);
			}
		}

		// Token: 0x0600004C RID: 76 RVA: 0x00006A84 File Offset: 0x00004C84
		private int GetPlayerPing(VRRig rig)
		{
			int num = this.TryGetRealPing(rig);
			bool flag = num > 0;
			int result;
			if (flag)
			{
				this.fakePing = -1;
				result = num;
			}
			else
			{
				result = this.GenerateFakePing();
			}
			return result;
		}

		// Token: 0x0600004D RID: 77 RVA: 0x00006AB8 File Offset: 0x00004CB8
		private void ConfirmDisconnect()
		{
			this.ResetSpeedTracking();
			PhotonNetwork.Disconnect();
		}

		// Token: 0x0600004E RID: 78 RVA: 0x00006AC8 File Offset: 0x00004CC8
		private void ResetSpeedTracking()
		{
			this.speedSampleRig = null;
			this.lastSpeedPos = Vector3.zero;
			this.lastSpeedTime = 0f;
			this.hasSpeedSample = false;
			this.selectedRig = null;
			this.lockedTarget = null;
			this.lastCheckedRig = null;
			this.currentCheaterPercent = 0f;
			this.displayedCheaterPercent = 0f;
			bool flag = this.speedMphText != null;
			if (flag)
			{
				this.speedMphText.text = "Speed mph: ---";
			}
		}

		// Token: 0x0600004F RID: 79 RVA: 0x00006B46 File Offset: 0x00004D46
		private IEnumerator ScalePopup(GameObject obj, Vector3 target, float time, Action onFinish = null)
		{
			Vector3 start = obj.transform.localScale;
			float t = 0f;
			while (t < 1f)
			{
				t += Time.deltaTime / time;
				obj.transform.localScale = Vector3.Lerp(start, target, Mathf.SmoothStep(0f, 1f, t));
				yield return null;
			}
			obj.transform.localScale = target;
			if (onFinish != null)
			{
				onFinish();
			}
			yield break;
		}

		// Token: 0x06000050 RID: 80 RVA: 0x00006B74 File Offset: 0x00004D74
		private int TryGetRealPing(VRRig rig)
		{
			int result;
			try
			{
				result = PhotonNetwork.NetworkingClient.LoadBalancingPeer.RoundTripTime;
			}
			catch
			{
				result = -1;
			}
			return result;
		}

		// Token: 0x06000051 RID: 81 RVA: 0x00006BAC File Offset: 0x00004DAC
		private int GenerateFakePing()
		{
			bool flag = this.fakePing < 0;
			if (flag)
			{
				int num = 40;
				try
				{
					num = PhotonNetwork.NetworkingClient.LoadBalancingPeer.RoundTripTime;
				}
				catch
				{
				}
				this.fakePing = num - this.pingRandom.Next(5, 20);
				bool flag2 = this.fakePing < 15;
				if (flag2)
				{
					this.fakePing = 15;
				}
			}
			this.fakePing += this.pingRandom.Next(-2, 3);
			bool flag3 = this.fakePing < 10;
			if (flag3)
			{
				this.fakePing = 10;
			}
			bool flag4 = Time.time >= this.nextSpikeTime;
			if (flag4)
			{
				int num2 = this.pingRandom.Next(15, 120);
				this.fakePing += num2;
				this.nextSpikeTime = Time.time + (float)this.pingRandom.Next(2, 6);
			}
			bool flag5 = this.fakePing > 70;
			if (flag5)
			{
				this.fakePing -= this.pingRandom.Next(1, 3);
			}
			return this.fakePing;
		}

		// Token: 0x06000052 RID: 82 RVA: 0x00006CDC File Offset: 0x00004EDC
		private void ResetSettings()
		{
			this.gunRayEnabled = true;
			this.autoLockEnabled = false;
			this.SetOpacity(0.95f);
			this.SetTheme(1);
			bool flag = this.gunRayToggle != null;
			if (flag)
			{
				this.gunRayToggle.GetComponent<Image>().color = this.activeGreen;
			}
			bool flag2 = this.autoLockToggle != null;
			if (flag2)
			{
				this.autoLockToggle.GetComponent<Image>().color = this.elementDark;
			}
			base.StartCoroutine(this.PlayRipple(this.resetBtn.GetComponent<Image>()));
		}

		// Token: 0x06000053 RID: 83 RVA: 0x00006D74 File Offset: 0x00004F74
		private GameObject CreateButton(Transform parent, string label, Vector2 pos, Action onClick)
		{
			GameObject gameObject = new GameObject(label);
			gameObject.transform.SetParent(parent, false);
			Image image = gameObject.AddComponent<Image>();
			image.sprite = this.CreateOptimizedRoundedSprite(512, 512, 6);
			image.type = Image.Type.Sliced;
			image.color = this.elementDark;
			RectTransform component = gameObject.GetComponent<RectTransform>();
			component.sizeDelta = new Vector2(90f, 26f);
			component.anchoredPosition = pos;
			GameObject gameObject2 = new GameObject("Text");
			gameObject2.transform.SetParent(gameObject.transform, false);
			TextMeshProUGUI textMeshProUGUI = gameObject2.AddComponent<TextMeshProUGUI>();
			textMeshProUGUI.text = label;
			textMeshProUGUI.fontSize = 9f;
			textMeshProUGUI.alignment = TextAlignmentOptions.Center;
			textMeshProUGUI.color = this.textLight;
			RectTransform component2 = gameObject2.GetComponent<RectTransform>();
			component2.anchorMin = Vector2.zero;
			component2.anchorMax = Vector2.one;
			component2.offsetMin = Vector2.zero;
			component2.offsetMax = Vector2.zero;
			Button button = gameObject.AddComponent<Button>();
			button.onClick.AddListener(delegate()
			{
				this.PlayClickFeedback();
				Action onClick2 = onClick;
				if (onClick2 != null)
				{
					onClick2();
				}
			});
			return gameObject;
		}

		// Token: 0x06000054 RID: 84 RVA: 0x00006EC4 File Offset: 0x000050C4
		private void ChangePage(int dir)
		{
			this.GoToPage(this.currentPage + dir);
		}

		// Token: 0x06000055 RID: 85 RVA: 0x00006ED8 File Offset: 0x000050D8
		private void UpdateInfoVisibility()
		{
			bool active = this.currentPage == 1;
			bool flag = this.playerNameText;
			if (flag)
			{
				this.playerNameText.gameObject.SetActive(active);
			}
			bool flag2 = this.playerColorCodeText;
			if (flag2)
			{
				this.playerColorCodeText.SetActive(active);
			}
			bool flag3 = this.playerPlatformText;
			if (flag3)
			{
				this.playerPlatformText.gameObject.SetActive(active);
			}
			bool flag4 = this.playerFPSText;
			if (flag4)
			{
				this.playerFPSText.gameObject.SetActive(active);
			}
			bool flag5 = this.playerPingText;
			if (flag5)
			{
				this.playerPingText.gameObject.SetActive(active);
			}
			bool flag6 = this.playerCreationDateText;
			if (flag6)
			{
				this.playerCreationDateText.gameObject.SetActive(active);
			}
			bool active2 = this.currentPage == 2;
			bool flag7 = this.playerModsText;
			if (flag7)
			{
				this.playerModsText.gameObject.SetActive(active2);
			}
			bool flag8 = this.playerCosmeticsText;
			if (flag8)
			{
				this.playerCosmeticsText.gameObject.SetActive(active2);
			}
			bool active3 = this.currentPage == 3;
			bool flag9 = this.importantChecksTitleText;
			if (flag9)
			{
				this.importantChecksTitleText.gameObject.SetActive(active3);
			}
			bool flag10 = this.speedMphText;
			if (flag10)
			{
				this.speedMphText.gameObject.SetActive(active3);
			}
			bool active4 = this.currentPage == 4;
			bool flag11 = this.lobbyCodeText;
			if (flag11)
			{
				this.lobbyCodeText.gameObject.SetActive(active4);
			}
			bool flag12 = this.lobbyPlayerCountText;
			if (flag12)
			{
				this.lobbyPlayerCountText.gameObject.SetActive(active4);
			}
			bool flag13 = this.lobbyGameModeText;
			if (flag13)
			{
				this.lobbyGameModeText.gameObject.SetActive(active4);
			}
			bool flag14 = this.lobbyQueueText;
			if (flag14)
			{
				this.lobbyQueueText.gameObject.SetActive(active4);
			}
			bool flag15 = this.lobbyMasterText;
			if (flag15)
			{
				this.lobbyMasterText.gameObject.SetActive(active4);
			}
			bool active5 = this.currentPage == 5;
			bool flag16 = this.gunRayToggle;
			if (flag16)
			{
				this.gunRayToggle.SetActive(active5);
			}
			bool flag17 = this.autoLockToggle;
			if (flag17)
			{
				this.autoLockToggle.SetActive(active5);
			}
			bool flag18 = this.opacityLowBtn;
			if (flag18)
			{
				this.opacityLowBtn.SetActive(active5);
			}
			bool flag19 = this.opacityMedBtn;
			if (flag19)
			{
				this.opacityMedBtn.SetActive(active5);
			}
			bool flag20 = this.opacityHighBtn;
			if (flag20)
			{
				this.opacityHighBtn.SetActive(active5);
			}
			bool flag21 = this.resetBtn;
			if (flag21)
			{
				this.resetBtn.SetActive(active5);
			}
			bool active6 = this.currentPage == 6;
			bool flag22 = this.reportBtn;
			if (flag22)
			{
				this.reportBtn.SetActive(active6);
			}
			bool flag23 = this.muteBtn;
			if (flag23)
			{
				this.muteBtn.SetActive(active6);
			}
			bool flag24 = this.blockBtn;
			if (flag24)
			{
				this.blockBtn.SetActive(active6);
			}
			bool flag25 = this.reportStatusText;
			if (flag25)
			{
				this.reportStatusText.gameObject.SetActive(active6);
			}
			bool active7 = this.currentPage == 7;
			bool flag26 = this.themeTitleText;
			if (flag26)
			{
				this.themeTitleText.gameObject.SetActive(active7);
			}
			foreach (GameObject gameObject in this.themeButtons)
			{
				bool flag27 = gameObject != null;
				if (flag27)
				{
					gameObject.SetActive(active7);
				}
			}
			bool active8 = this.currentPage == 8;
			bool flag28 = this.creditsOwnerText;
			if (flag28)
			{
				this.creditsOwnerText.gameObject.SetActive(active8);
			}
			bool flag29 = this.creditsCoOwnerText;
			if (flag29)
			{
				this.creditsCoOwnerText.gameObject.SetActive(active8);
			}
			bool flag30 = this.creditsHeadDevText;
			if (flag30)
			{
				this.creditsHeadDevText.gameObject.SetActive(active8);
			}
			bool flag31 = this.creditsDevText;
			if (flag31)
			{
				TextMeshProUGUI textMeshProUGUI = this.creditsDevText;
				if (textMeshProUGUI != null)
				{
					textMeshProUGUI.gameObject.SetActive(active8);
				}
			}
			bool flag32 = this.creditsThankYouText;
			if (flag32)
			{
				this.creditsThankYouText.gameObject.SetActive(active8);
			}
			bool flag33 = this.creditsText;
			if (flag33)
			{
				this.creditsText.gameObject.SetActive(active8);
			}
		}

		// Token: 0x06000056 RID: 86 RVA: 0x000073DC File Offset: 0x000055DC
		private void UpdatePage()
		{
			bool flag = this.selectedRig == null;
			if (!flag)
			{
				NetPlayer owningNetPlayer = this.selectedRig.OwningNetPlayer;
				string str = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null) ?? "Unknown";
				NetPlayer owningNetPlayer2 = this.selectedRig.OwningNetPlayer;
				string b = ((owningNetPlayer2 != null) ? owningNetPlayer2.UserId : null) ?? "";
				bool flag2 = false;
				for (int i = 0; i < this.ownerUserIds.Length; i++)
				{
					bool flag3 = this.ownerUserIds[i] == b;
					if (flag3)
					{
						flag2 = true;
						break;
					}
				}
				bool flag4 = flag2;
				if (flag4)
				{
					this.playerNameText.text = "<color=#FFD700>[OWNER]</color> <b>" + str + "</b>";
				}
				else
				{
					this.playerNameText.text = "<b>" + str + "</b>";
				}
				this.playerFPSText.text = "<b>FPS:</b> " + GTUtility.GetFPS(this.selectedRig);
				string text = GTUtility.GetPlatform(this.selectedRig);
				bool flag5 = string.IsNullOrEmpty(text) || text.ToLower().Contains("unknown");
				if (flag5)
				{
					string text2 = this.selectedRig.concatStringOfCosmeticsAllowed ?? "";
					string text3 = text2.ToLower();
					bool flag6 = text3.Contains("oculus") || text3.Contains("quest") || text3.Contains("oculusquest") || text3.Contains("oculus_store");
					if (flag6)
					{
						text = "Quest";
					}
					else
					{
						bool flag7 = text3.Contains("steam") || text3.Contains("steamvr");
						if (flag7)
						{
							text = "Steam";
						}
						else
						{
							bool flag8 = text3.Contains("rift") || text3.Contains("oculus rift");
							if (flag8)
							{
								text = "Rift";
							}
						}
					}
				}
				this.playerPlatformText.text = "<b>Platform:</b> " + text;
				bool flag9 = !string.IsNullOrEmpty(text) && text.ToLower().Contains("quest");
				if (flag9)
				{
					this.playerPlatformText.color = Color.green;
				}
				else
				{
					bool flag10 = !string.IsNullOrEmpty(text) && text.ToLower().Contains("steam");
					if (flag10)
					{
						this.playerPlatformText.color = Color.cyan;
					}
					else
					{
						this.playerPlatformText.color = this.textLight;
					}
				}
				bool flag11 = this.playerColorCodeText != null;
				if (flag11)
				{
					this.playerColorCodeText.GetComponent<TextMeshProUGUI>().text = "<b>Color:</b> " + this.GetColorDigits(this.selectedRig);
				}
				VRRig targetRig = this.selectedRig;
				bool flag12 = this.playerCreationDateText != null;
				if (flag12)
				{
					bool flag13 = targetRig == null || targetRig.OwningNetPlayer == null;
					if (flag13)
					{
						this.playerCreationDateText.text = "<b>Created:</b> UNKNOWN";
					}
					else
					{
						string targetUserId = targetRig.OwningNetPlayer.UserId;
						string accountCreationDate = GTUtility.GetAccountCreationDate(targetRig, delegate(object date)
						{
							bool flag16 = this.playerCreationDateText != null && this.selectedRig == targetRig && targetRig.OwningNetPlayer != null && targetRig.OwningNetPlayer.UserId == targetUserId;
							if (flag16)
							{
								this.playerCreationDateText.text = string.Format("<b>Created:</b> {0}", date);
							}
						});
						this.playerCreationDateText.text = "<b>Created:</b> " + accountCreationDate;
					}
				}
				this.playerModsText.text = "<b>Mods/Cheats:</b> " + GTUtility.GetCheats(this.selectedRig);
				this.playerCosmeticsText.text = "<b>Cosmetics:</b> " + GTUtility.GetCosmetics(this.selectedRig);
				int playerPing = this.GetPlayerPing(this.selectedRig);
				this.playerPingText.text = "<b>Ping:</b> " + ((playerPing >= 0) ? (playerPing.ToString() + " ms") : "???");
				bool flag14 = playerPing >= 0;
				if (flag14)
				{
					this.playerPingText.color = ((playerPing < 60) ? Color.green : ((playerPing < 120) ? Color.yellow : Color.red));
				}
				else
				{
					this.playerPingText.color = this.textLight;
				}
				bool flag15 = this.reportStatusText != null && this.currentPage == 6;
				if (flag15)
				{
					this.reportStatusText.text = "Selected: " + str;
				}
			}
		}

		// Token: 0x06000057 RID: 87 RVA: 0x00007884 File Offset: 0x00005A84
		private void UpdateSpeedDisplay(VRRig rig)
		{
			bool flag = this.speedMphText == null;
			if (!flag)
			{
				bool flag2 = rig == null;
				if (flag2)
				{
					this.speedMphText.text = "Speed mph: ---";
					this.speedSampleRig = null;
					this.hasSpeedSample = false;
				}
				else
				{
					bool flag3 = rig != this.speedSampleRig;
					if (flag3)
					{
						this.speedSampleRig = rig;
						Transform transform = this.GetSpeedAnchor(rig) ?? rig.transform;
						this.lastSpeedPos = transform.position;
						this.lastSpeedTime = Time.time - Mathf.Max(Time.deltaTime, 0.001f);
						this.hasSpeedSample = true;
					}
					Transform transform2 = this.GetSpeedAnchor(rig) ?? rig.transform;
					float num = Mathf.Max(Time.time - this.lastSpeedTime, 0.001f);
					Vector3 position = transform2.position;
					float num2 = Vector3.Distance(position, this.lastSpeedPos) / num;
					Rigidbody componentInParent = transform2.GetComponentInParent<Rigidbody>();
					bool flag4 = componentInParent != null;
					float num3;
					if (flag4)
					{
						Vector3 vector = Vector3.zero;
						PropertyInfo property = typeof(Rigidbody).GetProperty("linearVelocity", BindingFlags.Instance | BindingFlags.Public);
						bool flag5 = property != null;
						if (flag5)
						{
							try
							{
								vector = (Vector3)property.GetValue(componentInParent);
							}
							catch
							{
								vector = componentInParent.velocity;
							}
						}
						else
						{
							vector = componentInParent.velocity;
						}
						float magnitude = vector.magnitude;
						num3 = ((magnitude > 0.01f) ? magnitude : num2);
					}
					else
					{
						num3 = num2;
					}
					float num4 = num3 * 2.23694f;
					this.speedMphText.text = string.Format("Speed mph: {0:0.0}", num4);
					this.lastSpeedPos = position;
					this.lastSpeedTime = Time.time;
				}
			}
		}

		// Token: 0x06000058 RID: 88 RVA: 0x00007A5C File Offset: 0x00005C5C
		private Transform GetSpeedAnchor(VRRig rig)
		{
			bool flag = rig == null;
			Transform result;
			if (flag)
			{
				result = null;
			}
			else
			{
				try
				{
					FieldInfo field = typeof(VRRig).GetField("headMesh", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
					Transform transform = ((field != null) ? field.GetValue(rig) : null) as Transform;
					bool flag2 = transform != null;
					if (flag2)
					{
						return transform;
					}
				}
				catch
				{
				}
				foreach (Transform transform2 in rig.GetComponentsInChildren<Transform>())
				{
					string text = transform2.name.ToLowerInvariant();
					bool flag3 = text.Contains("head") || text.Contains("hand") || text.Contains("controller");
					if (flag3)
					{
						return transform2;
					}
				}
				result = rig.transform;
			}
			return result;
		}

		// Token: 0x06000059 RID: 89 RVA: 0x00007B44 File Offset: 0x00005D44
		private void CheckButtonTouch()
		{
			bool flag = this.uiPanel == null || !this.canPressButtons || this.uiCanvasGroup == null || !this.uiCanvasGroup.interactable;
			if (!flag)
			{
				Transform rightControllerTransform = this.GetRightControllerTransform();
				bool flag2 = rightControllerTransform == null;
				if (!flag2)
				{
					for (int i = 0; i < this.sidebarButtons.Count; i++)
					{
						int pageIndex = i;
						this.CheckTouch(rightControllerTransform, this.sidebarButtons[i], delegate
						{
							this.StartCoroutine(this.PlayRipple(this.sidebarButtonImages[pageIndex]));
							this.GoToPage(pageIndex + 1);
						});
					}
					bool flag3 = this.currentPage == 5;
					if (flag3)
					{
						this.CheckTouch(rightControllerTransform, this.gunRayToggle, delegate
						{
							this.ToggleGunRay();
						});
						this.CheckTouch(rightControllerTransform, this.autoLockToggle, delegate
						{
							this.ToggleAutoLock();
						});
						this.CheckTouch(rightControllerTransform, this.opacityLowBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.opacityLowBtn.GetComponent<Image>()));
							this.SetOpacity(0.7f);
						});
						this.CheckTouch(rightControllerTransform, this.opacityMedBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.opacityMedBtn.GetComponent<Image>()));
							this.SetOpacity(0.85f);
						});
						this.CheckTouch(rightControllerTransform, this.opacityHighBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.opacityHighBtn.GetComponent<Image>()));
							this.SetOpacity(0.95f);
						});
						this.CheckTouch(rightControllerTransform, this.resetBtn, delegate
						{
							this.ResetSettings();
						});
					}
					bool flag4 = this.currentPage == 6;
					if (flag4)
					{
						this.CheckTouch(rightControllerTransform, this.reportBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.reportBtn.GetComponent<Image>()));
							this.OpenReportConfirm();
						});
						this.CheckTouch(rightControllerTransform, this.muteBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.muteBtn.GetComponent<Image>()));
							this.OpenMuteConfirm();
						});
						this.CheckTouch(rightControllerTransform, this.blockBtn, delegate
						{
							base.StartCoroutine(this.PlayRipple(this.blockBtn.GetComponent<Image>()));
							this.OpenBlockConfirm();
						});
					}
					bool flag5 = this.currentPage == 7;
					if (flag5)
					{
						for (int j = 0; j < this.themeButtons.Count; j++)
						{
							int themeIdx = j;
							this.CheckTouch(rightControllerTransform, this.themeButtons[j], delegate
							{
								this.StartCoroutine(this.PlayRipple(this.themeButtons[themeIdx].GetComponent<Image>()));
								this.SetTheme(themeIdx);
							});
						}
					}
					this.CheckTouch(rightControllerTransform, this.disconnectBtn, delegate
					{
						this.OpenConfirmPopup("Leave the lobby?", new Action(this.ConfirmDisconnect));
					});
					bool flag6 = this.popupOpen;
					if (flag6)
					{
						this.CheckTouch(rightControllerTransform, this.confirmYesBtn, new Action(this.ExecutePendingAction));
						this.CheckTouch(rightControllerTransform, this.confirmNoBtn, new Action(this.CloseConfirmPopup));
					}
				}
			}
		}

		// Token: 0x0600005A RID: 90 RVA: 0x00007DD8 File Offset: 0x00005FD8
		private void CheckTouch(Transform hand, GameObject button, Action onTouch)
		{
			bool flag = button == null;
			if (!flag)
			{
				float num = Vector3.Distance(hand.position, button.transform.position);
				bool flag2 = num < 0.08f && this.canPressButtons;
				if (flag2)
				{
					base.StartCoroutine(this.ButtonCooldown());
					this.PlayClickFeedback();
					if (onTouch != null)
					{
						onTouch();
					}
				}
			}
		}

		// Token: 0x0600005B RID: 91 RVA: 0x00007E41 File Offset: 0x00006041
		private IEnumerator ButtonCooldown()
		{
			this.canPressButtons = false;
			yield return new WaitForSeconds(0.4f);
			this.canPressButtons = true;
			yield break;
		}

		// Token: 0x0600005C RID: 92 RVA: 0x00007E50 File Offset: 0x00006050
		private void PlayClickFeedback()
		{
			bool flag = false;
			try
			{
				bool flag2 = VRRig.LocalRig != null;
				if (flag2)
				{
					VRRig.LocalRig.PlayHandTapLocal(447, flag, 0.4f);
				}
			}
			catch
			{
			}
			try
			{
				bool flag3 = GorillaTagger.Instance != null;
				if (flag3)
				{
					float amplitude = GorillaTagger.Instance.tagHapticStrength / 2f;
					float duration = GorillaTagger.Instance.tagHapticDuration / 2f;
					GorillaTagger.Instance.StartVibration(flag, amplitude, duration);
				}
			}
			catch
			{
			}
		}

		// Token: 0x0600005D RID: 93 RVA: 0x00007EFC File Offset: 0x000060FC
		private IEnumerator PlayRipple(Image buttonImage)
		{
			bool flag = buttonImage == null;
			if (flag)
			{
				yield break;
			}
			GameObject ripple = new GameObject("Ripple");
			ripple.transform.SetParent(buttonImage.transform, false);
			Image rippleImg = ripple.AddComponent<Image>();
			rippleImg.color = new Color(1f, 1f, 1f, 0.12f);
			rippleImg.sprite = this.CreateOptimizedRoundedSprite(512, 512, 256);
			RectTransform rippleRect = ripple.GetComponent<RectTransform>();
			rippleRect.sizeDelta = Vector2.zero;
			float time = 0f;
			float duration = 0.38f;
			float maxSize = 96f;
			while (time < duration)
			{
				time += Time.deltaTime;
				float progress = time / duration;
				float size = Mathf.Lerp(0f, maxSize, progress);
				rippleRect.sizeDelta = new Vector2(size, size);
				rippleImg.color = new Color(1f, 1f, 1f, Mathf.Lerp(0.12f, 0f, progress));
				yield return null;
			}
			UnityEngine.Object.Destroy(ripple);
			yield break;
		}

		// Token: 0x0600005E RID: 94 RVA: 0x00007F14 File Offset: 0x00006114
		private void SetPanelAlpha(float alpha)
		{
			bool flag = this.uiPanel == null;
			if (!flag)
			{
				foreach (Graphic graphic in this.uiPanel.GetComponentsInChildren<Graphic>(true))
				{
					Color color = graphic.color;
					graphic.color = new Color(color.r, color.g, color.b, alpha);
				}
			}
		}

		// Token: 0x0600005F RID: 95 RVA: 0x00007F80 File Offset: 0x00006180
		private void FadeOutPanel()
		{
			bool flag = this.uiPanel == null;
			if (!flag)
			{
				bool flag2 = this.fadeRoutine != null;
				if (flag2)
				{
					base.StopCoroutine(this.fadeRoutine);
				}
				this.fadeRoutine = base.StartCoroutine(this.FadeOutAndDisable(this.uiPanel, 1f, 0f, 0.5f));
			}
		}

		// Token: 0x06000060 RID: 96 RVA: 0x00007FE1 File Offset: 0x000061E1
		private IEnumerator FadeOutAndDisable(GameObject root, float from, float to, float duration)
		{
			List<Graphic> graphics = new List<Graphic>(root.GetComponentsInChildren<Graphic>(true));
			float t = 0f;
			while (t < duration)
			{
				t += Time.deltaTime;
				float alpha = Mathf.Lerp(from, to, t / duration);
				foreach (Graphic g in graphics)
				{
					bool flag = g == null;
					if (!flag)
					{
						Color c = g.color;
						g.color = new Color(c.r, c.g, c.b, alpha);
						c = default(Color);
						g = null;
					}
				}
				List<Graphic>.Enumerator enumerator = default(List<Graphic>.Enumerator);
				yield return null;
			}
			foreach (Graphic g2 in graphics)
			{
				bool flag2 = g2 == null;
				if (!flag2)
				{
					Color c2 = g2.color;
					g2.color = new Color(c2.r, c2.g, c2.b, to);
					c2 = default(Color);
					g2 = null;
				}
			}
			List<Graphic>.Enumerator enumerator2 = default(List<Graphic>.Enumerator);
			this.ShowPanel(false);
			yield break;
		}

		// Token: 0x06000061 RID: 97 RVA: 0x0000800D File Offset: 0x0000620D
		private IEnumerator FadeAllGraphics(GameObject root, float from, float to, float duration, bool destroyAfter = false)
		{
			List<Graphic> graphics = new List<Graphic>(root.GetComponentsInChildren<Graphic>(true));
			float t = 0f;
			while (t < duration)
			{
				t += Time.deltaTime;
				float alpha = Mathf.Lerp(from, to, t / duration);
				foreach (Graphic g in graphics)
				{
					bool flag = g == null;
					if (!flag)
					{
						Color c = g.color;
						g.color = new Color(c.r, c.g, c.b, alpha);
						c = default(Color);
						g = null;
					}
				}
				List<Graphic>.Enumerator enumerator = default(List<Graphic>.Enumerator);
				yield return null;
			}
			foreach (Graphic g2 in graphics)
			{
				bool flag2 = g2 == null;
				if (!flag2)
				{
					Color c2 = g2.color;
					g2.color = new Color(c2.r, c2.g, c2.b, to);
					c2 = default(Color);
					g2 = null;
				}
			}
			List<Graphic>.Enumerator enumerator2 = default(List<Graphic>.Enumerator);
			if (destroyAfter)
			{
				root.SetActive(false);
			}
			yield break;
		}

		// Token: 0x06000062 RID: 98 RVA: 0x00008044 File Offset: 0x00006244
		private Sprite CreateOptimizedRoundedSprite(int width, int height, int cornerRadius)
		{
			Texture2D texture2D = new Texture2D(width, height, TextureFormat.ARGB32, false);
			texture2D.filterMode = FilterMode.Bilinear;
			texture2D.wrapMode = TextureWrapMode.Clamp;
			Color white = Color.white;
			float num = 1.5f;
			for (int i = 0; i < height; i++)
			{
				for (int j = 0; j < width; j++)
				{
					float a = 1f;
					bool flag = j < cornerRadius;
					bool flag2 = j >= width - cornerRadius;
					bool flag3 = i < cornerRadius;
					bool flag4 = i >= height - cornerRadius;
					bool flag5 = flag && flag3;
					if (flag5)
					{
						float num2 = Vector2.Distance(new Vector2((float)j, (float)i), new Vector2((float)cornerRadius, (float)cornerRadius));
						bool flag6 = num2 > (float)cornerRadius;
						if (flag6)
						{
							a = Mathf.Clamp01(1f - (num2 - (float)cornerRadius) / num);
						}
					}
					else
					{
						bool flag7 = flag2 && flag3;
						if (flag7)
						{
							float num3 = Vector2.Distance(new Vector2((float)j, (float)i), new Vector2((float)(width - cornerRadius - 1), (float)cornerRadius));
							bool flag8 = num3 > (float)cornerRadius;
							if (flag8)
							{
								a = Mathf.Clamp01(1f - (num3 - (float)cornerRadius) / num);
							}
						}
						else
						{
							bool flag9 = flag && flag4;
							if (flag9)
							{
								float num4 = Vector2.Distance(new Vector2((float)j, (float)i), new Vector2((float)cornerRadius, (float)(height - cornerRadius - 1)));
								bool flag10 = num4 > (float)cornerRadius;
								if (flag10)
								{
									a = Mathf.Clamp01(1f - (num4 - (float)cornerRadius) / num);
								}
							}
							else
							{
								bool flag11 = flag2 && flag4;
								if (flag11)
								{
									float num5 = Vector2.Distance(new Vector2((float)j, (float)i), new Vector2((float)(width - cornerRadius - 1), (float)(height - cornerRadius - 1)));
									bool flag12 = num5 > (float)cornerRadius;
									if (flag12)
									{
										a = Mathf.Clamp01(1f - (num5 - (float)cornerRadius) / num);
									}
								}
							}
						}
					}
					texture2D.SetPixel(j, i, new Color(1f, 1f, 1f, a));
				}
			}
			texture2D.Apply();
			return Sprite.Create(texture2D, new Rect(0f, 0f, (float)width, (float)height), new Vector2(0.5f, 0.5f), 100f, 1U, SpriteMeshType.FullRect, new Vector4((float)cornerRadius, (float)cornerRadius, (float)cornerRadius, (float)cornerRadius));
		}

		// Token: 0x06000063 RID: 99 RVA: 0x00008284 File Offset: 0x00006484
		private void UpdateGunRay()
		{
			Transform rightControllerTransform = this.GetRightControllerTransform();
			bool flag = rightControllerTransform == null;
			if (!flag)
			{
				bool flag2 = this.uiCanvasGroup == null || !this.uiCanvasGroup.interactable || this.uiPanel == null || !this.uiPanel.activeInHierarchy;
				if (flag2)
				{
					bool flag3 = this.gunRay != null;
					if (flag3)
					{
						UnityEngine.Object.Destroy(this.gunRay.gameObject);
						this.gunRay = null;
					}
					bool flag4 = this.gunSphere != null;
					if (flag4)
					{
						UnityEngine.Object.Destroy(this.gunSphere);
						this.gunSphere = null;
					}
				}
				else
				{
					bool flag5 = !SimpleInputs.RightGrab || !this.gunRayEnabled;
					if (flag5)
					{
						bool flag6 = this.gunRay != null;
						if (flag6)
						{
							UnityEngine.Object.Destroy(this.gunRay.gameObject);
							this.gunRay = null;
						}
						bool flag7 = this.gunSphere != null;
						if (flag7)
						{
							UnityEngine.Object.Destroy(this.gunSphere);
							this.gunSphere = null;
						}
					}
					else
					{
						bool flag8 = this.gunRay == null;
						if (flag8)
						{
							this.gunRay = new GameObject("GunRay").AddComponent<LineRenderer>();
							this.gunRay.startWidth = 0.012f;
							this.gunRay.endWidth = 0.012f;
							this.gunRay.material = new Material(Shader.Find("Sprites/Default"));
							this.gunRay.startColor = Color.lavender;
							this.gunRay.positionCount = 2;
						}
						bool flag9 = this.gunSphere == null;
						if (flag9)
						{
							this.gunSphere = GameObject.CreatePrimitive(PrimitiveType.Sphere);
							this.gunSphere.name = "GunEndSphere";
							this.gunSphere.transform.localScale = Vector3.one * 0.15f;
							Material material = new Material(Shader.Find("Unlit/Color"));
							material.color = new Color(0.5f, 0.55f, 0.7f, 1f);
							this.gunSphere.GetComponent<Renderer>().material = material;
							UnityEngine.Object.Destroy(this.gunSphere.GetComponent<Collider>());
						}
						Vector3 position = rightControllerTransform.position;
						Vector3 forward = rightControllerTransform.forward;
						int layerMask = ~LayerMask.GetMask(new string[]
						{
							"IgnoreRaycast"
						});
						Vector3 b = position + forward * 100f;
						bool flag10 = this.lockedTarget != null;
						if (flag10)
						{
							b = this.lockedTarget.position + Vector3.down * 0.3f;
						}
						else
						{
							RaycastHit raycastHit;
							bool flag11 = Physics.Raycast(position, forward, out raycastHit, 100f, layerMask);
							if (flag11)
							{
								b = raycastHit.point;
								VRRig componentInParent = raycastHit.collider.GetComponentInParent<VRRig>();
								bool flag12 = componentInParent != null && (SimpleInputs.RightTrigger || this.autoLockEnabled);
								if (flag12)
								{
									this.selectedRig = componentInParent;
									this.lockedTarget = componentInParent.transform;
									this.UpdatePage();
								}
								else
								{
									bool flag13 = componentInParent != null;
									if (flag13)
									{
										this.selectedRig = componentInParent;
									}
								}
							}
						}
						bool flag14 = !SimpleInputs.RightTrigger && !this.autoLockEnabled;
						if (flag14)
						{
							this.lockedTarget = null;
						}
						this.gunRay.SetPosition(0, position);
						Vector3 a = (this.gunRay.positionCount > 1) ? this.gunRay.GetPosition(1) : (position + forward * 100f);
						this.gunRay.SetPosition(1, Vector3.Lerp(a, b, Time.deltaTime * 15f));
						this.gunSphere.transform.position = Vector3.Lerp(this.gunSphere.transform.position, b, Time.deltaTime * 15f);
					}
				}
			}
		}

		// Token: 0x06000064 RID: 100 RVA: 0x00008690 File Offset: 0x00006890
		private IEnumerator PublishLocalPingRoutine()
		{
			for (;;)
			{
				int myPing = -1;
				try
				{
					myPing = PhotonNetwork.GetPing();
				}
				catch
				{
					try
					{
						myPing = PhotonNetwork.NetworkingClient.LoadBalancingPeer.RoundTripTime;
					}
					catch
					{
						myPing = -1;
					}
				}
				try
				{
					ExitGames.Client.Photon.Hashtable tbl = new ExitGames.Client.Photon.Hashtable();
					tbl["Ping"] = myPing;
					Player localPlayer = PhotonNetwork.LocalPlayer;
					if (localPlayer != null)
					{
						localPlayer.SetCustomProperties(tbl, null, null);
					}
					tbl = null;
				}
				catch
				{
				}
				yield return new WaitForSeconds(5f);
			}
			yield break;
		}

		// Token: 0x06000065 RID: 101 RVA: 0x000086A0 File Offset: 0x000068A0
		private string GetPingDisplay(int ping)
		{
			bool flag = ping < 0;
			string result;
			if (flag)
			{
				result = "<color=#888888>???</color>";
			}
			else
			{
				bool flag2 = ping <= 60;
				string arg;
				string arg2;
				if (flag2)
				{
					arg = "#00FF00";
					arg2 = "||||";
				}
				else
				{
					bool flag3 = ping <= 120;
					if (flag3)
					{
						arg = "#FFFF00";
						arg2 = "|||.";
					}
					else
					{
						bool flag4 = ping <= 200;
						if (flag4)
						{
							arg = "#FFA500";
							arg2 = "||..";
						}
						else
						{
							arg = "#FF0000";
							arg2 = "|...";
						}
					}
				}
				result = string.Format("<b><color={0}>{1} ms  {2}</color></b>", arg, ping, arg2);
			}
			return result;
		}

		// Token: 0x06000066 RID: 102 RVA: 0x00008744 File Offset: 0x00006944
		private string GetColorDigits(VRRig rig)
		{
			bool flag = rig == null;
			string result;
			if (flag)
			{
				result = "---";
			}
			else
			{
				int num = Mathf.Clamp(Mathf.RoundToInt(rig.playerColor.r * 9f), 0, 9);
				int num2 = Mathf.Clamp(Mathf.RoundToInt(rig.playerColor.g * 9f), 0, 9);
				int num3 = Mathf.Clamp(Mathf.RoundToInt(rig.playerColor.b * 9f), 0, 9);
				result = string.Format("{0} {1} {2}", num, num2, num3);
			}
			return result;
		}

		// Token: 0x06000067 RID: 103 RVA: 0x000087E8 File Offset: 0x000069E8
		private void SetTimeOfDaySafe(int timeIndex)
		{
			bool flag = BetterDayNightManager.instance == null;
			if (!flag)
			{
				BetterDayNightManager.instance.SetTimeOfDay(timeIndex);
			}
		}

		// Token: 0x06000068 RID: 104 RVA: 0x00008818 File Offset: 0x00006A18
		private void UpdateLobbyInfo()
		{
			bool flag = this.currentPage != 4;
			if (!flag)
			{
				bool flag2 = this.lobbyCodeText == null;
				if (!flag2)
				{
					bool flag3 = !PhotonNetwork.InRoom || PhotonNetwork.CurrentRoom == null;
					if (flag3)
					{
						this.lobbyCodeText.text = "Lobby: Not in room";
						this.lobbyPlayerCountText.text = "Players: ---";
						this.lobbyGameModeText.text = "Game Mode: ---";
						this.lobbyQueueText.text = "Queue: ---";
						this.lobbyMasterText.text = "Master: ---";
					}
					else
					{
						Room currentRoom = PhotonNetwork.CurrentRoom;
						this.lobbyCodeText.text = "Lobby: " + currentRoom.Name;
						this.lobbyPlayerCountText.text = string.Format("Players: {0}/{1}", currentRoom.PlayerCount, currentRoom.MaxPlayers);
						string roomProp = this.GetRoomProp(currentRoom, new string[]
						{
							"gameMode",
							"gamemode",
							"mode",
							"GameMode"
						}, "Unknown");
						string str = this.ExtractKeyword(roomProp, new string[]
						{
							"superinfection",
							"infection",
							"casual",
							"tag",
							"hunt",
							"brawl",
							"paintbrawl",
							"default"
						}, "Unknown");
						this.lobbyGameModeText.text = "Game Mode: " + str;
						string roomProp2 = this.GetRoomProp(currentRoom, new string[]
						{
							"queue",
							"Queue"
						}, "Default");
						string str2 = this.ExtractKeyword(roomProp2, new string[]
						{
							"competitive",
							"minigames",
							"default",
							"casual"
						}, roomProp2);
						this.lobbyQueueText.text = "Queue: " + str2;
						string str3 = (PhotonNetwork.MasterClient != null) ? PhotonNetwork.MasterClient.NickName : "Unknown";
						this.lobbyMasterText.text = "Master: " + str3;
					}
				}
			}
		}

		// Token: 0x06000069 RID: 105 RVA: 0x00008A50 File Offset: 0x00006C50
		private string GetRoomProp(Room room, string[] keys, string fallback)
		{
			bool flag = room == null || room.CustomProperties == null;
			string result;
			if (flag)
			{
				result = fallback;
			}
			else
			{
				foreach (string key in keys)
				{
					bool flag2 = room.CustomProperties.ContainsKey(key);
					if (flag2)
					{
						object obj = room.CustomProperties[key];
						bool flag3 = obj != null;
						if (flag3)
						{
							return obj.ToString();
						}
					}
				}
				result = fallback;
			}
			return result;
		}

		// Token: 0x0600006A RID: 106 RVA: 0x00008ACC File Offset: 0x00006CCC
		private string CleanLastWord(string raw, string fallback)
		{
			bool flag = string.IsNullOrWhiteSpace(raw);
			string result;
			if (flag)
			{
				result = fallback;
			}
			else
			{
				int num = raw.Length - 1;
				while (num >= 0 && !char.IsLetter(raw[num]))
				{
					num--;
				}
				bool flag2 = num < 0;
				if (flag2)
				{
					result = fallback;
				}
				else
				{
					int num2 = num;
					while (num2 >= 0 && char.IsLetter(raw[num2]))
					{
						num2--;
					}
					string text = raw.Substring(num2 + 1, num - num2);
					result = (string.IsNullOrWhiteSpace(text) ? fallback : text);
				}
			}
			return result;
		}

		// Token: 0x0600006B RID: 107 RVA: 0x00008B64 File Offset: 0x00006D64
		private string ExtractKeyword(string raw, string[] known, string fallback)
		{
			bool flag = string.IsNullOrWhiteSpace(raw);
			string result;
			if (flag)
			{
				result = fallback;
			}
			else
			{
				string text = raw.ToLowerInvariant();
				string text2 = null;
				foreach (string text3 in known)
				{
					int num = text.LastIndexOf(text3.ToLowerInvariant());
					bool flag2 = num >= 0;
					if (flag2)
					{
						text2 = raw.Substring(num, text3.Length);
					}
				}
				bool flag3 = !string.IsNullOrWhiteSpace(text2);
				if (flag3)
				{
					result = text2;
				}
				else
				{
					result = this.CleanLastWord(raw, fallback);
				}
			}
			return result;
		}

		// Token: 0x0600006C RID: 108 RVA: 0x00008BF8 File Offset: 0x00006DF8
		private void UpdateUIPanelPosition()
		{
			bool flag = this.uiPanel == null;
			if (!flag)
			{
				Transform leftControllerTransform = this.GetLeftControllerTransform();
				bool flag2 = leftControllerTransform != null;
				if (flag2)
				{
					this.uiPanel.transform.position = leftControllerTransform.position + leftControllerTransform.right * 0.09f - leftControllerTransform.up * 0.02f + leftControllerTransform.forward * 0.03f;
					this.uiPanel.transform.rotation = leftControllerTransform.rotation * Quaternion.Euler(125f, 0f, 10f);
				}
			}
		}

		// Token: 0x0600006D RID: 109 RVA: 0x00008CB8 File Offset: 0x00006EB8
		private Transform GetRightControllerTransform()
		{
			bool flag = GTPlayer.Instance == null;
			Transform result;
			if (flag)
			{
				result = null;
			}
			else
			{
				Transform transform = GTPlayer.Instance.transform.Find("rig/body/shoulder.R/upper_arm.R/forearm.R/hand.R");
				bool flag2 = transform != null;
				if (flag2)
				{
					result = transform;
				}
				else
				{
					foreach (Transform transform2 in GTPlayer.Instance.GetComponentsInChildren<Transform>())
					{
						string text = transform2.name.ToLowerInvariant();
						bool flag3 = text.Contains("righthand") || text.Contains("right_hand") || text == "hand.r" || text == "right hand";
						if (flag3)
						{
							return transform2;
						}
					}
					result = null;
				}
			}
			return result;
		}

		// Token: 0x0600006E RID: 110 RVA: 0x00008D80 File Offset: 0x00006F80
		private Transform GetLeftControllerTransform()
		{
			bool flag = GTPlayer.Instance == null;
			Transform result;
			if (flag)
			{
				result = null;
			}
			else
			{
				Transform transform = GTPlayer.Instance.transform.Find("rig/body/shoulder.L/upper_arm.L/forearm.L/hand.L");
				bool flag2 = transform != null;
				if (flag2)
				{
					result = transform;
				}
				else
				{
					foreach (Transform transform2 in GTPlayer.Instance.GetComponentsInChildren<Transform>())
					{
						string text = transform2.name.ToLowerInvariant();
						bool flag3 = text.Contains("lefthand") || text.Contains("left_hand") || text == "hand.l" || text == "left hand";
						if (flag3)
						{
							return transform2;
						}
					}
					result = null;
				}
			}
			return result;
		}

		// Token: 0x04000008 RID: 8
		private bool lastXButtonState = false;

		// Token: 0x04000009 RID: 9
		private GameObject uiPanel;

		// Token: 0x0400000A RID: 10
		private Coroutine fadeRoutine;

		// Token: 0x0400000B RID: 11
		private bool canPressButtons = true;

		// Token: 0x0400000C RID: 12
		private const float TOUCH_DISTANCE = 0.08f;

		// Token: 0x0400000D RID: 13
		private TextMeshProUGUI titleText;

		// Token: 0x0400000E RID: 14
		private TextMeshProUGUI pageText;

		// Token: 0x0400000F RID: 15
		private TextMeshProUGUI pageNameText;

		// Token: 0x04000010 RID: 16
		private TextMeshProUGUI importantChecksTitleText;

		// Token: 0x04000011 RID: 17
		private TextMeshProUGUI speedMphText;

		// Token: 0x04000012 RID: 18
		private TextMeshProUGUI playerNameText;

		// Token: 0x04000013 RID: 19
		private TextMeshProUGUI playerPlatformText;

		// Token: 0x04000014 RID: 20
		private TextMeshProUGUI playerFPSText;

		// Token: 0x04000015 RID: 21
		private TextMeshProUGUI playerCreationDateText;

		// Token: 0x04000016 RID: 22
		private TextMeshProUGUI playerModsText;

		// Token: 0x04000017 RID: 23
		private TextMeshProUGUI playerCosmeticsText;

		// Token: 0x04000018 RID: 24
		private GameObject gunRayToggle;

		// Token: 0x04000019 RID: 25
		private GameObject autoLockToggle;

		// Token: 0x0400001A RID: 26
		private GameObject opacityLowBtn;

		// Token: 0x0400001B RID: 27
		private GameObject opacityMedBtn;

		// Token: 0x0400001C RID: 28
		private GameObject opacityHighBtn;

		// Token: 0x0400001D RID: 29
		private GameObject resetBtn;

		// Token: 0x0400001E RID: 30
		private TextMeshProUGUI creditsText;

		// Token: 0x0400001F RID: 31
		private TextMeshProUGUI creditsOwnerText;

		// Token: 0x04000020 RID: 32
		private TextMeshProUGUI creditsCoOwnerText;

		// Token: 0x04000021 RID: 33
		private TextMeshProUGUI creditsHeadDevText;

		// Token: 0x04000022 RID: 34
		private TextMeshProUGUI creditsDevText;

		// Token: 0x04000023 RID: 35
		private TextMeshProUGUI creditsThankYouText;

		// Token: 0x04000024 RID: 36
		private TextMeshProUGUI playerPingText;

		// Token: 0x04000025 RID: 37
		private GameObject separatorLine;

		// Token: 0x04000026 RID: 38
		private TextMeshProUGUI lobbyCodeText;

		// Token: 0x04000027 RID: 39
		private TextMeshProUGUI lobbyPlayerCountText;

		// Token: 0x04000028 RID: 40
		private TextMeshProUGUI lobbyGameModeText;

		// Token: 0x04000029 RID: 41
		private TextMeshProUGUI lobbyQueueText;

		// Token: 0x0400002A RID: 42
		private TextMeshProUGUI lobbyMasterText;

		// Token: 0x0400002B RID: 43
		private GameObject reportBtn;

		// Token: 0x0400002C RID: 44
		private GameObject muteBtn;

		// Token: 0x0400002D RID: 45
		private GameObject blockBtn;

		// Token: 0x0400002E RID: 46
		private TextMeshProUGUI reportStatusText;

		// Token: 0x0400002F RID: 47
		private List<GameObject> themeButtons = new List<GameObject>();

		// Token: 0x04000030 RID: 48
		private TextMeshProUGUI themeTitleText;

		// Token: 0x04000031 RID: 49
		private LineRenderer gunRay;

		// Token: 0x04000032 RID: 50
		private GameObject gunSphere;

		// Token: 0x04000033 RID: 51
		private VRRig selectedRig;

		// Token: 0x04000034 RID: 52
		private Transform lockedTarget;

		// Token: 0x04000035 RID: 53
		private bool gunRayEnabled = true;

		// Token: 0x04000036 RID: 54
		private bool autoLockEnabled = false;

		// Token: 0x04000037 RID: 55
		private float menuOpacity = 0.95f;

		// Token: 0x04000038 RID: 56
		private int currentPage = 1;

		// Token: 0x04000039 RID: 57
		private int maxPage = 8;

		// Token: 0x0400003A RID: 58
		private VRRig speedSampleRig;

		// Token: 0x0400003B RID: 59
		private Vector3 lastSpeedPos;

		// Token: 0x0400003C RID: 60
		private float lastSpeedTime;

		// Token: 0x0400003D RID: 61
		private bool hasSpeedSample;

		// Token: 0x0400003E RID: 62
		private int currentTheme = 1;

		// Token: 0x0400003F RID: 63
		private Color panelColor;

		// Token: 0x04000040 RID: 64
		private Color elementDark;

		// Token: 0x04000041 RID: 65
		private Color textLight;

		// Token: 0x04000042 RID: 66
		private Color separatorColor;

		// Token: 0x04000043 RID: 67
		private Color activeGreen;

		// Token: 0x04000044 RID: 68
		private Color accentColor;

		// Token: 0x04000045 RID: 69
		private Color sidebarColor;

		// Token: 0x04000046 RID: 70
		private Color sidebarBtnColor;

		// Token: 0x04000047 RID: 71
		private Color sidebarBtnActiveColor;

		// Token: 0x04000048 RID: 72
		private Color cheaterPanelColor;

		// Token: 0x04000049 RID: 73
		private Color cheaterPanelOutline;

		// Token: 0x0400004A RID: 74
		private Color fadeOutColor;

		// Token: 0x0400004B RID: 75
		private readonly Color[][] themePresets = new Color[][]
		{
			new Color[]
			{
				new Color(0.38f, 0.32f, 0.52f, 0.95f),
				new Color(0.26f, 0.22f, 0.4f, 1f),
				new Color(0.93f, 0.91f, 0.97f, 1f),
				new Color(0.55f, 0.48f, 0.7f, 1f),
				new Color(0.58f, 0.52f, 0.85f, 1f),
				new Color(0.6f, 0.5f, 0.85f, 0.75f),
				new Color(0.22f, 0.18f, 0.32f, 0.95f),
				new Color(0.3f, 0.25f, 0.45f, 1f),
				new Color(0.58f, 0.52f, 0.85f, 1f),
				new Color(0.15f, 0.12f, 0.22f, 0.95f),
				new Color(0.9f, 0.25f, 0.3f, 0.6f),
				new Color(0.5f, 0.35f, 0.6f, 0f)
			},
			new Color[]
			{
				new Color(0.15f, 0.22f, 0.35f, 0.95f),
				new Color(0.1f, 0.16f, 0.28f, 1f),
				new Color(0.92f, 0.95f, 1f, 1f),
				new Color(0.3f, 0.45f, 0.65f, 1f),
				new Color(0.25f, 0.55f, 0.85f, 1f),
				new Color(0.3f, 0.6f, 0.9f, 0.75f),
				new Color(0.08f, 0.12f, 0.22f, 0.95f),
				new Color(0.12f, 0.2f, 0.35f, 1f),
				new Color(0.25f, 0.55f, 0.85f, 1f),
				new Color(0.06f, 0.1f, 0.18f, 0.95f),
				new Color(0.25f, 0.55f, 0.85f, 0.6f),
				new Color(0.2f, 0.35f, 0.55f, 0f)
			},
			new Color[]
			{
				new Color(0.18f, 0.32f, 0.22f, 0.95f),
				new Color(0.12f, 0.24f, 0.16f, 1f),
				new Color(0.9f, 1f, 0.92f, 1f),
				new Color(0.3f, 0.5f, 0.35f, 1f),
				new Color(0.35f, 0.7f, 0.45f, 1f),
				new Color(0.4f, 0.75f, 0.5f, 0.75f),
				new Color(0.1f, 0.2f, 0.14f, 0.95f),
				new Color(0.14f, 0.28f, 0.18f, 1f),
				new Color(0.35f, 0.7f, 0.45f, 1f),
				new Color(0.08f, 0.16f, 0.1f, 0.95f),
				new Color(0.35f, 0.7f, 0.45f, 0.6f),
				new Color(0.25f, 0.45f, 0.3f, 0f)
			},
			new Color[]
			{
				new Color(0.38f, 0.18f, 0.2f, 0.95f),
				new Color(0.28f, 0.12f, 0.14f, 1f),
				new Color(1f, 0.92f, 0.92f, 1f),
				new Color(0.55f, 0.3f, 0.32f, 1f),
				new Color(0.75f, 0.35f, 0.38f, 1f),
				new Color(0.8f, 0.4f, 0.42f, 0.75f),
				new Color(0.22f, 0.1f, 0.12f, 0.95f),
				new Color(0.32f, 0.15f, 0.17f, 1f),
				new Color(0.75f, 0.35f, 0.38f, 1f),
				new Color(0.18f, 0.08f, 0.1f, 0.95f),
				new Color(0.75f, 0.35f, 0.38f, 0.6f),
				new Color(0.5f, 0.25f, 0.28f, 0f)
			}
		};

		// Token: 0x0400004C RID: 76
		private readonly string[] themeNames = new string[]
		{
			"Purple",
			"Blue",
			"Green",
			"Red"
		};

		// Token: 0x0400004D RID: 77
		private readonly string[] pageNames = new string[]
		{
			"Player Info",
			"Mods & Cosmetics",
			"Speed Check",
			"Lobby Info",
			"Panel Settings",
			"Report & Mute",
			"Themes",
			"Credits"
		};

		// Token: 0x0400004E RID: 78
		private readonly string[] pageIcons = new string[]
		{
			"\ud83d\udc64",
			"\ud83c\udfae",
			"⚡",
			"\ud83c\udfe0",
			"⚙",
			"⚠",
			"\ud83c\udfa8",
			"★"
		};

		// Token: 0x0400004F RID: 79
		private const int clickSoundId = 447;

		// Token: 0x04000050 RID: 80
		private const float clickSoundVolume = 0.4f;

		// Token: 0x04000051 RID: 81
		private System.Random pingRandom = new System.Random();

		// Token: 0x04000052 RID: 82
		private int fakePing = -1;

		// Token: 0x04000053 RID: 83
		private float nextSpikeTime = 0f;

		// Token: 0x04000054 RID: 84
		private GameObject disconnectBtn;

		// Token: 0x04000055 RID: 85
		private GameObject confirmPopup;

		// Token: 0x04000056 RID: 86
		private GameObject confirmYesBtn;

		// Token: 0x04000057 RID: 87
		private GameObject confirmNoBtn;

		// Token: 0x04000058 RID: 88
		private TextMeshProUGUI confirmMessageText;

		// Token: 0x04000059 RID: 89
		private bool popupOpen = false;

		// Token: 0x0400005A RID: 90
		private Action pendingConfirmAction;

		// Token: 0x0400005B RID: 91
		private GameObject playerColorCodeText;

		// Token: 0x0400005C RID: 92
		private GameObject cheaterPanel;

		// Token: 0x0400005D RID: 93
		private TextMeshProUGUI cheaterTitleText;

		// Token: 0x0400005E RID: 94
		private TextMeshProUGUI cheaterPercentText;

		// Token: 0x0400005F RID: 95
		private GameObject cheaterBarBg;

		// Token: 0x04000060 RID: 96
		private GameObject cheaterBarFill;

		// Token: 0x04000061 RID: 97
		private Image cheaterBarFillImage;

		// Token: 0x04000062 RID: 98
		private Image cheaterPanelImage;

		// Token: 0x04000063 RID: 99
		private Outline cheaterPanelOutlineComp;

		// Token: 0x04000064 RID: 100
		private float cheaterBarWidth = 140f;

		// Token: 0x04000065 RID: 101
		private float currentCheaterPercent = 0f;

		// Token: 0x04000066 RID: 102
		private VRRig lastCheckedRig = null;

		// Token: 0x04000067 RID: 103
		private float displayedCheaterPercent = 0f;

		// Token: 0x04000068 RID: 104
		private GameObject sidebarPanel;

		// Token: 0x04000069 RID: 105
		private Image sidebarBgImage;

		// Token: 0x0400006A RID: 106
		private List<GameObject> sidebarButtons = new List<GameObject>();

		// Token: 0x0400006B RID: 107
		private List<Image> sidebarButtonImages = new List<Image>();

		// Token: 0x0400006C RID: 108
		private readonly string[] ownerUserIds = new string[]
		{
			"tlinesid",
			"daddypandasid",
			"thareonsid",
			"holdensid"
		};

		// Token: 0x0400006D RID: 109
		private CanvasGroup uiCanvasGroup;

		// Token: 0x0400006E RID: 110
		private Image mainBgImage;

		// Token: 0x0400006F RID: 111
		private List<Image> allThemedButtons = new List<Image>();

		// Token: 0x04000070 RID: 112
		private List<Image> allActionButtons = new List<Image>();

		// Token: 0x04000071 RID: 113
		private Dictionary<string, Sprite> downloadedIcons = new Dictionary<string, Sprite>();

		// Token: 0x04000072 RID: 114
		private readonly Dictionary<string, string> iconUrls = new Dictionary<string, string>
		{
			{
				"home",
				"https://i.ibb.co/FqdC7ccy/house-11.png"
			},
			{
				"speed",
				"https://i.ibb.co/y9PZ4bz/gauge.png"
			},
			{
				"mods",
				"https://i.ibb.co/mrJBJyyf/file-box.png"
			},
			{
				"report",
				"https://i.ibb.co/tkHg39F/circle-alert.png"
			},
			{
				"account",
				"https://i.ibb.co/vpXq8cp/user.png"
			},
			{
				"credits",
				"https://i.ibb.co/qMJtsJbw/star.png"
			},
			{
				"settings",
				"https://i.ibb.co/Mx04tdsv/cog-1.png"
			},
			{
				"themes",
				"https://i.ibb.co/Kz0k6zHp/palette-1.png"
			}
		};

		// Token: 0x04000073 RID: 115
		private readonly string[] pageIconKeys = new string[]
		{
			"account",
			"mods",
			"speed",
			"home",
			"settings",
			"report",
			"themes",
			"credits"
		};

		// Token: 0x04000074 RID: 116
		private List<Image> sidebarIconImages = new List<Image>();

		// Token: 0x04000075 RID: 117
		private Coroutine creationDateRefreshRoutine;

		// Token: 0x04000076 RID: 118
		private Coroutine publishPingRoutine;

		// Token: 0x04000077 RID: 119
		private string lastRoom = "";
	}
}

using System;

// Token: 0x02000007 RID: 7
internal class SimpleInputs
{
	// Token: 0x17000002 RID: 2
	// (get) Token: 0x06000016 RID: 22 RVA: 0x00002E6F File Offset: 0x0000106F
	public static bool RightTrigger
	{
		get
		{
			return ControllerInputPoller.instance.rightControllerIndexFloat > 0.5f;
		}
	}

	// Token: 0x17000003 RID: 3
	// (get) Token: 0x06000017 RID: 23 RVA: 0x00002E84 File Offset: 0x00001084
	public static bool RightGrab
	{
		get
		{
			return ControllerInputPoller.instance.rightGrab;
		}
	}

	// Token: 0x17000004 RID: 4
	// (get) Token: 0x06000018 RID: 24 RVA: 0x00002E92 File Offset: 0x00001092
	public static bool RightA
	{
		get
		{
			return ControllerInputPoller.instance.rightControllerSecondaryButton;
		}
	}

	// Token: 0x17000005 RID: 5
	// (get) Token: 0x06000019 RID: 25 RVA: 0x00002EA0 File Offset: 0x000010A0
	public static bool RightB
	{
		get
		{
			return ControllerInputPoller.instance.rightControllerSecondaryButton;
		}
	}

	// Token: 0x17000006 RID: 6
	// (get) Token: 0x0600001A RID: 26 RVA: 0x00002EAE File Offset: 0x000010AE
	public static bool LeftTrigger
	{
		get
		{
			return ControllerInputPoller.instance.leftControllerIndexFloat > 0.5f;
		}
	}

	// Token: 0x17000007 RID: 7
	// (get) Token: 0x0600001B RID: 27 RVA: 0x00002EC3 File Offset: 0x000010C3
	public static bool LeftGrab
	{
		get
		{
			return ControllerInputPoller.instance.leftGrab;
		}
	}

	// Token: 0x17000008 RID: 8
	// (get) Token: 0x0600001C RID: 28 RVA: 0x00002ED1 File Offset: 0x000010D1
	public static bool LeftX
	{
		get
		{
			return ControllerInputPoller.instance.leftControllerPrimaryButton;
		}
	}

	// Token: 0x17000009 RID: 9
	// (get) Token: 0x0600001D RID: 29 RVA: 0x00002EDF File Offset: 0x000010DF
	public static bool LeftY
	{
		get
		{
			return ControllerInputPoller.instance.leftControllerSecondaryButton;
		}
	}
}

using System;
using System.Collections.Generic;
using HarmonyLib;
using UnityEngine;

// Token: 0x02000006 RID: 6
public class PlayerFPSMonitor : MonoBehaviour
{
	// Token: 0x06000011 RID: 17 RVA: 0x00002D10 File Offset: 0x00000F10
	public static List<VRRig> GetAllRemoteRigs()
	{
		List<VRRig> list = new List<VRRig>();
		foreach (VRRig vrrig in UnityEngine.Object.FindObjectsByType<VRRig>(FindObjectsSortMode.None))
		{
			bool flag = vrrig == null;
			if (!flag)
			{
				bool flag2 = !vrrig.isOfflineVRRig;
				if (flag2)
				{
					list.Add(vrrig);
				}
			}
		}
		return list;
	}

	// Token: 0x06000012 RID: 18 RVA: 0x00002D70 File Offset: 0x00000F70
	public static VRRig GetRigByIndex(int index)
	{
		List<VRRig> allRemoteRigs = PlayerFPSMonitor.GetAllRemoteRigs();
		bool flag = allRemoteRigs.Count == 0 || index < 0 || index >= allRemoteRigs.Count;
		VRRig result;
		if (flag)
		{
			result = null;
		}
		else
		{
			result = allRemoteRigs[index];
		}
		return result;
	}

	// Token: 0x06000013 RID: 19 RVA: 0x00002DB4 File Offset: 0x00000FB4
	public static int GetFPS(VRRig rig)
	{
		bool flag = rig == null;
		int result;
		if (flag)
		{
			result = -1;
		}
		else
		{
			Traverse traverse = Traverse.Create(rig).Field("fps");
			bool flag2 = traverse == null;
			if (flag2)
			{
				result = -1;
			}
			else
			{
				object value = traverse.GetValue();
				bool flag3 = value == null;
				if (flag3)
				{
					result = -1;
				}
				else
				{
					result = (int)value;
				}
			}
		}
		return result;
	}

	// Token: 0x06000014 RID: 20 RVA: 0x00002E10 File Offset: 0x00001010
	private void Update()
	{
		VRRig rigByIndex = PlayerFPSMonitor.GetRigByIndex(this.selectedIndex);
		bool flag = rigByIndex != null;
		if (flag)
		{
			int fps = PlayerFPSMonitor.GetFPS(rigByIndex);
			Debug.Log(string.Format("Player {0} FPS = {1}", this.selectedIndex, fps));
		}
	}

	// Token: 0x04000007 RID: 7
	public int selectedIndex = 0;
}

using System;
using UnityEngine;

// Token: 0x02000005 RID: 5
public class PlayerFPS : MonoBehaviour
{
	// Token: 0x17000001 RID: 1
	// (get) Token: 0x0600000D RID: 13 RVA: 0x00002CB0 File Offset: 0x00000EB0
	// (set) Token: 0x0600000E RID: 14 RVA: 0x00002CB8 File Offset: 0x00000EB8
	public int CurrentFPS { get; private set; }

	// Token: 0x0600000F RID: 15 RVA: 0x00002CC1 File Offset: 0x00000EC1
	private void Update()
	{
		this.deltaTime += (Time.unscaledDeltaTime - this.deltaTime) * 0.1f;
		this.CurrentFPS = Mathf.RoundToInt(1f / this.deltaTime);
	}

	// Token: 0x04000006 RID: 6
	private float deltaTime = 0f;
}

using System;
using System.Collections;
using System.Collections.Generic;
using GorillaLocomotion;
using GorillaNetworking;
using HarmonyLib;
using PlayFab;
using PlayFab.ClientModels;
using UnityEngine;

// Token: 0x02000004 RID: 4
public static class GTUtility
{
	// Token: 0x06000003 RID: 3 RVA: 0x00002069 File Offset: 0x00000269
	public static VRRig GetLocalVRRig()
	{
		GTPlayer instance = GTPlayer.Instance;
		return (instance != null) ? instance.GetComponentInChildren<VRRig>() : null;
	}

	// Token: 0x06000004 RID: 4 RVA: 0x0000207C File Offset: 0x0000027C
	public static string GetPlayerName()
	{
		VRRig localVRRig = GTUtility.GetLocalVRRig();
		string text;
		if (localVRRig == null)
		{
			text = null;
		}
		else
		{
			NetPlayer owningNetPlayer = localVRRig.OwningNetPlayer;
			text = ((owningNetPlayer != null) ? owningNetPlayer.NickName : null);
		}
		return text ?? "Unknown";
	}

	// Token: 0x06000005 RID: 5 RVA: 0x000020A4 File Offset: 0x000002A4
	public static string GetUserId(VRRig rig)
	{
		string text;
		if (rig == null)
		{
			text = null;
		}
		else
		{
			NetPlayer owningNetPlayer = rig.OwningNetPlayer;
			text = ((owningNetPlayer != null) ? owningNetPlayer.UserId : null);
		}
		return text ?? "Unknown";
	}

	// Token: 0x06000006 RID: 6 RVA: 0x000020C8 File Offset: 0x000002C8
	public static string GetAccountCreationDate(VRRig rig, Action<object> value)
	{
		bool flag = rig == null || rig.OwningNetPlayer == null;
		string result2;
		if (flag)
		{
			result2 = "UNKNOWN";
		}
		else
		{
			string userId = rig.OwningNetPlayer.UserId;
			bool flag2 = GTUtility.datePool.ContainsKey(userId);
			if (flag2)
			{
				result2 = GTUtility.datePool[userId];
			}
			else
			{
				GTUtility.datePool[userId] = "LOADING";
				PlayFabClientAPI.GetAccountInfo(new GetAccountInfoRequest
				{
					PlayFabId = userId
				}, delegate(GetAccountInfoResult result)
				{
					string value2 = result.AccountInfo.Created.ToString("MMM dd, yyyy HH:mm").ToUpperInvariant();
					GTUtility.datePool[userId] = value2;
					rig.UpdateName();
				}, delegate(PlayFabError error)
				{
					GTUtility.datePool[userId] = "ERROR";
					rig.UpdateName();
				}, null, null);
				result2 = "LOADING";
			}
		}
		return result2;
	}

	// Token: 0x06000007 RID: 7 RVA: 0x000021A0 File Offset: 0x000003A0
	public static string GetCheats(VRRig rig)
	{
		bool flag = rig == null || rig.OwningNetPlayer == null;
		string result;
		if (flag)
		{
			result = "None";
		}
		else
		{
			NetPlayer owningNetPlayer = rig.OwningNetPlayer;
			Dictionary<string, object> dictionary = new Dictionary<string, object>();
			foreach (DictionaryEntry dictionaryEntry in owningNetPlayer.GetPlayerRef().CustomProperties)
			{
				dictionary[dictionaryEntry.Key.ToString().ToLower()] = dictionaryEntry.Value;
			}
			string text = "";
			foreach (KeyValuePair<string, string> keyValuePair in GTUtility.specialModsList)
			{
				bool flag2 = dictionary.ContainsKey(keyValuePair.Key.ToLower());
				if (flag2)
				{
					text = text + ((text == "") ? "" : ", ") + keyValuePair.Value.ToUpper();
				}
			}
			result = (string.IsNullOrEmpty(text) ? "None" : text);
		}
		return result;
	}

	// Token: 0x06000008 RID: 8 RVA: 0x000022E8 File Offset: 0x000004E8
	public static string GetFPS(VRRig rig)
	{
		bool flag = rig == null;
		bool flag2 = flag;
		string result;
		if (flag2)
		{
			result = "N/A";
		}
		else
		{
			Traverse traverse = Traverse.Create(rig).Field("fps");
			bool flag3 = traverse != null;
			bool flag4 = flag3;
			if (flag4)
			{
				result = "FPS " + traverse.GetValue().ToString();
			}
			else
			{
				result = "N/A";
			}
		}
		return result;
	}

	// Token: 0x06000009 RID: 9 RVA: 0x00002358 File Offset: 0x00000558
	public static string GetPlatform(VRRig rig)
	{
		bool flag = rig == null;
		bool flag2 = flag;
		string result;
		if (flag2)
		{
			result = "STANDALONE";
		}
		else
		{
			string concatStringOfCosmeticsAllowed = rig.concatStringOfCosmeticsAllowed;
			bool flag3 = concatStringOfCosmeticsAllowed.Contains("game-purchase-bundle");
			bool flag4 = flag3;
			if (flag4)
			{
				result = "RIFT";
			}
			else
			{
				bool flag5 = concatStringOfCosmeticsAllowed.Contains("S. FIRST LOGIN");
				bool flag6 = flag5;
				if (flag6)
				{
					result = "STEAM";
				}
				else
				{
					bool flag7 = concatStringOfCosmeticsAllowed.Contains("FIRST LOGIN");
					bool flag8 = flag7;
					if (flag8)
					{
						result = "PC";
					}
					else
					{
						result = "STANDALONE";
					}
				}
			}
		}
		return result;
	}

	// Token: 0x0600000A RID: 10 RVA: 0x000023F8 File Offset: 0x000005F8
	public static List<VRRig> GetAllPlayers()
	{
		VRRig[] collection = UnityEngine.Object.FindObjectsByType<VRRig>(FindObjectsSortMode.None);
		return new List<VRRig>(collection);
	}

	// Token: 0x0600000B RID: 11 RVA: 0x00002418 File Offset: 0x00000618
	public static string GetCosmetics(VRRig rig)
	{
		bool flag = rig == null;
		string result;
		if (flag)
		{
			result = "None";
		}
		else
		{
			string text = "";
			string text2 = rig.concatStringOfCosmeticsAllowed ?? "";
			foreach (KeyValuePair<string, string[]> keyValuePair in GTUtility.SpecialCosmetics)
			{
				bool flag2 = !string.IsNullOrEmpty(text2) && text2.Contains(keyValuePair.Key);
				if (flag2)
				{
					text = string.Concat(new string[]
					{
						text,
						(text == "") ? "" : ", ",
						"<color=#",
						keyValuePair.Value[1],
						">",
						keyValuePair.Value[0],
						"</color>"
					});
				}
			}
			bool flag3 = rig.cosmeticSet != null && !string.IsNullOrEmpty(text2);
			if (flag3)
			{
				foreach (CosmeticsController.CosmeticItem cosmeticItem in rig.cosmeticSet.items)
				{
					bool flag4 = !cosmeticItem.isNullItem && text2.IndexOf(cosmeticItem.itemName, StringComparison.OrdinalIgnoreCase) < 0;
					if (flag4)
					{
						text = text + ((text == "") ? "" : ", ") + "<color=red>COSMETX</color>";
						break;
					}
				}
			}
			result = ((text == "") ? "None" : text);
		}
		return result;
	}

	// Token: 0x04000002 RID: 2
	private static readonly Dictionary<string, string> datePool = new Dictionary<string, string>();

	// Token: 0x04000003 RID: 3
	private static Dictionary<string, string> specialModsList = new Dictionary<string, string>
	{
		{
			"genesis",
			"GENESIS"
		},
		{
			"HP_Left",
			"HOLDABLEPAD"
		},
		{
			"GrateVersion",
			"GRATE"
		},
		{
			"void",
			"VOID"
		},
		{
			"BANANAOS",
			"BANANAOS"
		},
		{
			"GC",
			"GORILLACRAFT"
		},
		{
			"CarName",
			"GORILLAVEHICLES"
		},
		{
			"6p72ly3j85pau2g9mda6ib8px",
			"CCMV2"
		},
		{
			"FPS-Nametags for Zlothy",
			"FPSTAGS"
		},
		{
			"cronos",
			"CRONOS"
		},
		{
			"ORBIT",
			"ORBIT"
		},
		{
			"Violet On Top",
			"VIOLET"
		},
		{
			"MP25",
			"MONKEPHONE"
		},
		{
			"GorillaWatch",
			"GORILLAWATCH"
		},
		{
			"InfoWatch",
			"GORILLAINFOWATCH"
		},
		{
			"BananaPhone",
			"BANANAPHONE"
		},
		{
			"Vivid",
			"VIVID"
		},
		{
			"RGBA",
			"CUSTOMCOSMETICS"
		},
		{
			"cheese is gouda",
			"WHOSICHEATING"
		},
		{
			"shirtversion",
			"GORILLASHIRTS"
		},
		{
			"gpronouns",
			"GORILLAPRONOUNS"
		},
		{
			"gfaces",
			"GORILLAFACES"
		},
		{
			"monkephone",
			"MONKEPHONE"
		},
		{
			"pmversion",
			"PLAYERMODELS"
		},
		{
			"gtrials",
			"GORILLATRIALS"
		},
		{
			"msp",
			"MONKESMARTPHONE"
		},
		{
			"gorillastats",
			"GORILLASTATS"
		},
		{
			"using gorilladrift",
			"GORILLADRIFT"
		},
		{
			"monkehavocversion",
			"MONKEHAVOC"
		},
		{
			"tictactoe",
			"TICTACTOE"
		},
		{
			"ccolor",
			"INDEX"
		},
		{
			"imposter",
			"GORILLAAMONGUS"
		},
		{
			"spectapeversion",
			"SPECTAPE"
		},
		{
			"cats",
			"CATS"
		},
		{
			"made by biotest05 :3",
			"DOGS"
		},
		{
			"fys cool magic mod",
			"FYSMAGICMOD"
		},
		{
			"colour",
			"CUSTOMCOSMETICS"
		},
		{
			"chainedtogether",
			"CHAINED TOGETHER"
		},
		{
			"goofywalkversion",
			"GOOFYWALK"
		},
		{
			"void_menu_open",
			"VOID"
		},
		{
			"violetpaiduser",
			"VIOLETPAID"
		},
		{
			"violetfree",
			"VIOLETFREE"
		},
		{
			"obsidianmc",
			"OBSIDIAN.LOL"
		},
		{
			"dark",
			"SHIBAGT DARK"
		},
		{
			"hidden menu",
			"HIDDEN"
		},
		{
			"oblivionuser",
			"OBLIVION"
		},
		{
			"hgrehngio889584739_hugb\n",
			"RESURGENCE"
		},
		{
			"eyerock reborn",
			"EYEROCK"
		},
		{
			"asteroidlite",
			"ASTEROID LITE"
		},
		{
			"elux",
			"ELUX"
		},
		{
			"cokecosmetics",
			"COKE COSMETX"
		},
		{
			"GFaces",
			"gFACES"
		},
		{
			"github.com/maroon-shadow/SimpleBoards",
			"SIMPLEBOARDS"
		},
		{
			"ObsidianMC",
			"OBSIDIAN"
		},
		{
			"hgrehngio889584739_hugb",
			"RESURGENCE"
		},
		{
			"GTrials",
			"gTRIALS"
		},
		{
			"github.com/ZlothY29IQ/GorillaMediaDisplay",
			"GMD"
		},
		{
			"github.com/ZlothY29IQ/TooMuchInfo",
			"TOOMUCHINFO"
		},
		{
			"github.com/ZlothY29IQ/RoomUtils-IW",
			"ROOMUTILS-IW"
		},
		{
			"github.com/ZlothY29IQ/MonkeClick",
			"MONKECLICK"
		},
		{
			"github.com/ZlothY29IQ/MonkeClick-CI",
			"MONKECLICK-CI"
		},
		{
			"github.com/ZlothY29IQ/MonkeRealism",
			"MONKEREALISM"
		},
		{
			"MediaPad",
			"MEDIAPAD"
		},
		{
			"GorillaCinema",
			"gCINEMA"
		},
		{
			"ChainedTogetherActive",
			"CHAINEDTOGETHER"
		},
		{
			"GPronouns",
			"gPRONOUNS"
		},
		{
			"CSVersion",
			"CustomSkin"
		},
		{
			"github.com/ZlothY29IQ/Zloth-RecRoomRig",
			"ZLOTH-RRR"
		},
		{
			"ShirtProperties",
			"SHIRTS-OLD"
		},
		{
			"GorillaShirts",
			"SHIRTS"
		},
		{
			"GS",
			"OLD SHIRTS"
		},
		{
			"6XpyykmrCthKhFeUfkYGxv7xnXpoe2",
			"CCMV2"
		},
		{
			"Body Tracking",
			"BODYTRACK-OLD"
		},
		{
			"Body Estimation",
			"HANBodyEst"
		},
		{
			"Gorilla Track",
			"BODYTRACK"
		},
		{
			"CustomMaterial",
			"CUSTOMCOSMETICS"
		},
		{
			"I like cheese",
			"RECROOMRIG"
		},
		{
			"silliness",
			"SILLINESS"
		},
		{
			"emotewheel",
			"EMOTEWHEEL"
		},
		{
			"untitled",
			"UNTITLED"
		},
		{
			"OculusReportMenu",
			"ReportMenu"
		},
		{
			"CastingShouldBeFree",
			"FreeCastingMod"
		},
		{
			"Elixir",
			"ElixirMenu"
		},
		{
			"ThareonsChecker",
			"ThareonsChecker"
		},
		{
			"MonkeyNotificationLib",
			"JarvisMod (Possibility)"
		},
		{
			"RexonPAID",
			"RexonPaid"
		},
		{
			"ui",
			"ZelixModChecker (M)"
		},
		{
			"",
			"Thareons UI"
		},
		{
			"FemBoyChecker",
			"Sentinel Menu"
		}
	};

	// Token: 0x04000004 RID: 4
	public static readonly Dictionary<string, string[]> SpecialCosmetics = new Dictionary<string, string[]>
	{
		{
			"LBAAD.",
			new string[]
			{
				"ADMINISTRATOR",
				"FF0000"
			}
		},
		{
			"LBAAK.",
			new string[]
			{
				"FOREST GUIDE",
				"867556"
			}
		},
		{
			"LBADE.",
			new string[]
			{
				"FINGER PAINTER",
				"00FF00"
			}
		},
		{
			"LBAGS.",
			new string[]
			{
				"ILLUSTRATOR",
				"C76417"
			}
		},
		{
			"LMAPY.",
			new string[]
			{
				"FOREST GUIDE MOD STICK"
			}
		},
		{
			"LBANI.",
			new string[]
			{
				"AA CREATOR BADGE"
			}
		},
		{
			"LHAAC.",
			new string[]
			{
				"Party Hat"
			}
		}
	};
}

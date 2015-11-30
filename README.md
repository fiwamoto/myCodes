using System;
using System.Collections.Generic;
using System.Linq;
using System.Linq.Expressions;
using System.Reflection;
using System.Web;
using System.Web.Mvc;
using ARCA.Core;
using ARCA.Domain.Enums;
using ARCA.Domain.Helpers;
using ARCA.Domain.Infrastructure;
using ARCA.Web.Models;

namespace ARCA.Web.Controllers
{
    public abstract class BaseController<T> : Controller where T : class 
    {
        public List<T> OrderList(List<T> genericList, string jtSorting)
        {
            string direction = jtSorting.Split(' ')[1];
            string field = jtSorting.Split(' ')[0];

            if (direction == "ASC")
            {
                return genericList.OrderBy(x => x.GetType().GetProperty(field).GetValue(x, null)).ToList();
            }
            return genericList.OrderByDescending(x => x.GetType().GetProperty(field).GetValue(x, null)).ToList();
        }

        public List<T> PageList(List<T> genericList, int jtStartIndex, int jtPageSize)
        {
            return genericList.Skip(jtStartIndex).Take(jtPageSize).ToList();
        }

        public void GenerateExcel(List<T> genericList, string excelFileName, ExportToExcelType reportType)
        {
            var sl = ARCAModules.GetInstance<IExportToExcel<T>>().Export(genericList, reportType);
            Response.ClearContent();
            Response.Clear();
            string responseHeaderValue = string.Format("attachment; filename={0}-{1}.xlsx", excelFileName, DateTime.Now);
            Response.AppendHeader("content-disposition", responseHeaderValue);
            Response.Charset = "";
            Response.Cache.SetCacheability(HttpCacheability.NoCache);
            Response.ContentType = "application/vnd.ms-excel";

            sl.SaveAs(Response.OutputStream);

            Response.End();
        }

        [HttpPost]
        public JsonResult GetSearchableFields()
        {
            var result = ARCAModules.GetInstance<IGeneral>().GetSearchableProperties<T>();
            return Json(new { Result = result });
        }

        [HttpPost]
        public int IncludeSearchableField(string propertyName, string propertyValue)
        {
            if (Session["searchConditions"] == null)
            {
                Session["searchConditions"] = new List<IdLinqExpression<T>>();
            }

            if (Session["searchConditionId"] == null)
            {
                Session["searchConditionId"] = 0;
            }

            var searchConditionList = (List<IdLinqExpression<T>>)Session["searchConditions"];
            var searchConditionId = (int)Session["searchConditionId"];

            var condition = ARCAModules.GetInstance<IGeneral>().GenerateLambdaExpression<T>(propertyName, propertyValue);

            searchConditionList.Add(new IdLinqExpression<T>(searchConditionId, condition));

            Session["searchConditions"] = searchConditionList;
            Session["searchConditionId"] = searchConditionId + 1;

            return searchConditionId;
        }

        public ActionResult FiltersAddedPartialView(int index, string field, string value)
        {
            var propertyDisplayName = ARCAModules.GetInstance<IGeneral>().GetPropertyDisplayName<T>(field);
            return PartialView("~/Views/Shared/FiltersAdded.cshtml", new FiltersAddedViewModel()
            {
                Index = index,
                Field = propertyDisplayName,
                Value = value
            });
        }

        public List<T> ApplySearchCriteria(List<T> genericList)
        {
            if (Session["searchConditions"] == null)
            {
                return genericList;
            }

            Expression<Func<T, bool>> finalExpression = null;

            foreach (var expression in (List<IdLinqExpression<T>>)Session["searchConditions"])
            {
                finalExpression = finalExpression == null ? expression.LinqExpression : finalExpression.And(expression.LinqExpression);
            }

            if (finalExpression != null)
                genericList = genericList.AsQueryable().Where(finalExpression).ToList();

            return genericList;
        }

        public void RemoveSearchableField(int index)
        {
            if (Session["searchConditions"] == null)
            {
                return;
            }

            var searchConditionList = (List<IdLinqExpression<T>>)Session["searchConditions"];

            var i = searchConditionList.TakeWhile(searchCondition => searchCondition.Id != index).Count();

            searchConditionList.RemoveAt(i);

            Session["searchConditions"] = searchConditionList;
        }

        public void RemoveAllSearchableFields()
        {
            if (Session["searchConditions"] == null)
            {
                return;
            }

            Session["searchConditions"] = null;
        }
    }
}
